#### 1.prepack vs webpack的说明
今天facebook开源了一个[prepack](https://github.com/facebook/prepack)，当时就很好奇。它到底和webpack之间的关系是什么？于是各种google,最后还是去官网上看了下各种例子。例子都很好理解，但是对于其和webpack的关系还是有点迷糊。最后找到了一个好用的插件，即[prepack-webpack-plugin](https://github.com/gajus/prepack-webpack-plugin),这才恍然大悟~

#### 2.解析prepack-webpack-plugin源码
下面我们直接给出这个插件的apply源码，因为webpack的plugin的所有逻辑都是在apply方法中处理的。内容如下:
```js
import ModuleFilenameHelpers from 'webpack/lib/ModuleFilenameHelpers';
import {
  RawSource
} from 'webpack-sources';
import {
  prepack
} from 'prepack';
import type {
  PluginConfigurationType,
  UserPluginConfigurationType
} from './types';
const defaultConfiguration = {
  prepack: {},
  test: /\.js($|\?)/i
};
export default class PrepackPlugin {
  configuration: PluginConfigurationType;
  constructor (userConfiguration?: UserPluginConfigurationType) {
    this.configuration = {
      ...defaultConfiguration,
      ...userConfiguration
    };
  }
  apply (compiler: Object) {
    const configuration = this.configuration;
    compiler.plugin('compilation', (compilation) => {
      compilation.plugin('optimize-chunk-assets', (chunks, callback) => {
        for (const chunk of chunks) {
          const files = chunk.files;
          //chunk.files获取该chunk产生的所有的输出文件，记住是输出文件
          for (const file of files) {
            const matchObjectConfiguration = {
              test: configuration.test
            };
            if (!ModuleFilenameHelpers.matchObject(matchObjectConfiguration, file)) {
              // eslint-disable-next-line no-continue
              continue;
            }
            const asset = compilation.assets[file];
            //获取文件本身
            const code = asset.source();
            //获取文件的代码内容
            const prepackedCode = prepack(code, {
              ...configuration.prepack,
              filename: file
            });
            //所以，这里是在webpack打包后对ES5代码的处理
            compilation.assets[file] = new RawSource(prepackedCode.code);
          }
        }
        callback();
      });
    });
  }
}
```
首先对于webpack各种钩子函数时机不了解的可以[点击这里](https://github.com/liangklfangl/webpack-compiler-and-compilation)。如果对于webpack中各个对象的属性不了解的可以[点击这里](https://github.com/liangklfangl/webpack-common-sense)。接下来我们对上面的代码进行简单的剖析:

(1)首先看for循环的前面那几句
```js
  const files = chunk.files;
  //chunk.files获取该chunk产生的所有的输出文件，记住是输出文件
  for (const file of files) {
   //这里我们只会对该chunk包含的文件中符合test规则的文件进行后续处理
    const matchObjectConfiguration = {
      test: configuration.test
    };
    if (!ModuleFilenameHelpers.matchObject(matchObjectConfiguration, file)) {
      // eslint-disable-next-line no-continue
      continue;
    }
}
```
我们这里也给出ModuleFilenameHelpers.matchObject的代码:
```js
//将字符串转化为regex
function asRegExp(test) {
    if(typeof test === "string") test = new RegExp("^" + test.replace(/[-[\]{}()*+?.,\\^$|#\s]/g, "\\$&"));
    return test;
}
ModuleFilenameHelpers.matchPart = function matchPart(str, test) {
    if(!test) return true;
    test = asRegExp(test);
    if(Array.isArray(test)) {
        return test.map(asRegExp).filter(function(regExp) {
            return regExp.test(str);
        }).length > 0;
    } else {
        return test.test(str);
    }
};
ModuleFilenameHelpers.matchObject = function matchObject(obj, str) {
    if(obj.test)
        if(!ModuleFilenameHelpers.matchPart(str, obj.test)) 
        return false;
    //获取test，如果这个文件名称符合test规则返回true，否则为false
    if(obj.include)
        if(!ModuleFilenameHelpers.matchPart(str, obj.include)) return false;
    if(obj.exclude)
        if(ModuleFilenameHelpers.matchPart(str, obj.exclude)) return false;
    return true;
};
```
这几句代码是一目了然的，如果这个产生的文件名称符合test规则返回true，否则为false。

(2)我们继续看后面对于符合规则的文件的处理
```js
 //如果满足规则我们继续处理~
 const asset = compilation.assets[file];
//获取编译产生的资源
const code = asset.source();
//获取文件的代码内容
const prepackedCode = prepack(code, {
  ...configuration.prepack,
  filename: file
});
//所以，这里是在webpack打包后对ES5代码的处理
compilation.assets[file] = new RawSource(prepackedCode.code);
```
其中asset.source表示的是模块的内容,你可以[点击这里](https://github.com/liangklfangl/webpack-common-sense)查看。假如我们的模块是一个html,内容如下:
```html
<header class="header">{{text}}</header>
```
最后打包的结果为:
```js
module.exports = "<header class=\\"header\\">{{text}}</header>";' }
```
这也是为什么我们会有下面的代码:
```js
compilation.assets[basename] = {
      source: function () {
        return results.source;
      },
      //source是文件的内容，通过fs.readFileAsync完成
      size: function () {
        return results.size.size;
        //size通过 fs.statAsync(filename)完成
      }
    };
    return basename;
  });
```
前面两句代码我们都分析过了，我们继续看下面的内容:
```js
const prepackedCode = prepack(code, {
  ...configuration.prepack,
  filename: file
});
//所以，这里是在webpack打包后对ES5代码的处理
compilation.assets[file] = new RawSource(prepackedCode.code);
```
此时才真正的对webpack打包后的代码进行处理，prepack的nodejs用法可以[查看这里](https://prepack.io/getting-started.html)。最后一句代码其实就是操作我们的输出资源，在输出资源中添加一个文件，文件的内容就是prepack打包后的代码。其中webpack-source的内容你可以[点击这里](https://github.com/webpack/webpack-sources)。按照官方的说明，该对象可以获取源代码，hash,内容大小，sourceMap等所有信息。我们给出对RowSourceMap的说明:
<pre>
RawSource
Represents source code without SourceMap.
new RawSource(sourceCode: String)
</pre>

很显然，就是显示源代码而不包含sourceMap。

#### 3.总结
所以，从我的理解来说，prepack作用于webpack的时机在于：将源代码转化为ES5以后。从上面的html的编译结果就可以知道了，至于它到底做了什么，以及如何做的，还请查看[官网](https://prepack.io/getting-started.html)。所以，如果你对webpack的基础知识不了解，请仔细查看我在文章中提到的那些内容。最后，**All issue and star welcomed!**

