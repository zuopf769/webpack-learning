## Webpack中的静态资源文件指纹

> [左鹏飞](https://github.com/zuopf769)  2017.09.30


本文讲解了在webpack中如何给静态资源加hash值：每次构建过程都会生成一个新的hash，所以一般用于做版本控制；chunkhash是基于内容生成的，但是webpack把所有类型的文件都以js为汇聚点打成一个bundle，改了css也会导致整个js的hash发生改变，所以最好通过ExtractTextWebpackPlugin把css独立抽取出来；chunkhash只能用于动态导入的chunk,这样每次build，入口静态导入的文件还是会生成新的hash,所以还需要webpack-md5-hash插件来完善该功能。


### 1. 如何添加文件名添加指纹

文件的hash指纹是前端静态资源实现增量更新最常用的方案。

在Webpack编译输出文件名（output.filename）的配置中，对于单个入口起点，filename 会是一个静态名称。

当通过多个入口起点(entry point)、代码拆分(code splitting)或各种插件(plugin)创建多个bundle，提供了多种模板字符串替换方式，来赋予每个 bundle 一个唯一的名称。

如果需要为文件加入hash指纹，Webpack提供了两个模板字符串可供使用：hash和chunkhash。


代码示例如下：

```
// 使用每次构建过程中，唯一的 hash 生成
filename: "[name].[hash].bundle.js"

```

```
// 使用基于每个 chunk 内容的 hash：
filename: "[chunkhash].bundle.js"
```

### 2. hash和chunkhash

#### hash

我在文档中找到了几个处关于hash的定义：

+  `Using the unique hash generated for every build`;
+  `[hash] in this parameter will be replaced with an hash of the compilation`
+  `The hash of the module identifier`

翻译过来就是每个构建过程生成的唯一hash。

#### chunkhash

chunkhash的解释是：`Using hashes based on each chunks' content`; 翻译过来就是基于每个chunk的内容而生成的hash。


### 3. chunk

chunk在Webpack中的含义，简单讲就是模块。

> + chunk的解释还是在[1.0的文档][chunk](http://webpack.github.io/docs/code-splitting.html)中能更深刻的理解。

> + `output.chunkFilename`决定了非入口(non-entry) chunk 文件的名称。 从这句话中也可以看出来chunk就是模块；只不过模块又分入口chunk文件和按需动态加载的chunk。


### 4. compilation和compiler

Webpack官方文档中[How to write a plugin](https://webpack.js.org/development/how-to-write-a-plugin/)章节有对compilation的详解。

> A compilation object represents a single build of versioned assets. While running webpack development middleware, a new compilation will be created each time a file change is detected, thus generating a new set of compiled assets. A compilation surfaces information about the present state of module resources, compiled assets, changed files, and watched dependencies. The compilation also provides many callback points at which a plugin may choose to perform custom actions.

compilation对象代表某个版本的资源对应的编译进程。当使用Webpack的development中间件时，每次检测到项目文件有改动就会创建一个compilation，进而能够针对改动生产全新的编译文件。compilation对象包含当前模块资源、待编译文件、有改动的文件和监听依赖的所有信息。


与compilation对应的有个compiler对象，通过对比，可以帮助大家对compilation有更深入的理解。

> The compiler object represents the fully configured Webpack environment. This object is built once upon starting Webpack, and is configured with all operational settings including options, loaders, and plugins.

compiler对象代表的是配置完备的Webpack环境。 compiler对象只在Webpack启动时构建一次，由Webpack组合所有的配置项构建生成。


简单的讲，compiler对象代表的是不变的webpack环境，是针对webpack的；而compilation对象针对的是随时可变的项目文件，只要文件有改动，compilation就会被重新创建。


### 5. hash的问题

理解了compilation之后，再回头看hash的定义：

[hash] is replaced by the hash of the compilation.

compilation在项目中任何一个文件改动后就会被重新创建，然后webpack计算新的compilation的hash值，这个hash值便是hash。

```
entry: {
    index: './src/index.js',
    print: './src/print.js'
},
```

```
output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[hash].js',
    publicPath: '/'
},
```

第一次build的结果：

![](https://github.com/zuopf769/webpack-learning/blob/master/Webpack%E4%B8%AD%E7%9A%84%E9%9D%99%E6%80%81%E8%B5%84%E6%BA%90%E6%96%87%E4%BB%B6%E6%8C%87%E7%BA%B9/images/1.png)

> 所有的文件名都会使用相同的hash指纹

第二次build的结果

![](https://github.com/zuopf769/webpack-learning/blob/master/Webpack%E4%B8%AD%E7%9A%84%E9%9D%99%E6%80%81%E8%B5%84%E6%BA%90%E6%96%87%E4%BB%B6%E6%8C%87%E7%BA%B9/images/2.png)

> 2个js文件任何一个改动都会影响另外1个文件的最终文件名。上线后，另外1个文件的浏览器缓存也全部失效。这肯定不是我们想要的结果。


那么如何避免这个问题呢？答案就是chunkhash！



### 6.  chunkhash怎么使用

`output.filename`不会影响那些「按需加载 chunk」的输出文件。对于这些文件，需要使用 `output.chunkFilename `选项来控制输出。

根据chunkhash的定义知道，chunkhash是根据具体模块文件的内容计算所得的hash值，所以某个文件的改动只会影响它本身的hash指纹，不会影响其他文件。

配置webpack的output如下：

```
// webpack.config.js配置
output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name][hash].js',
    chunkFilename: '[name][chunkhash].js',
    publicPath: '/'
}
```

```
// index.js动态引入chunk
import(/* webpackChunkName: "lodash" */ 'lodash').then(function(_) {
    console.log(_);
}).catch();
```

build的结果：

![](https://github.com/zuopf769/webpack-learning/blob/master/Webpack%E4%B8%AD%E7%9A%84%E9%9D%99%E6%80%81%E8%B5%84%E6%BA%90%E6%96%87%E4%BB%B6%E6%8C%87%E7%BA%B9/images/3.png)

每个文件的hash指纹都不相同，上线后无改动的文件不会失去缓存。


### 7. hash应用场景

接上文所述，webpack的hash字段是根据每次编译compilation的内容计算所得，也可以理解为项目总体文件的hash值，而不是针对每个具体文件的。

webpack针对compilation提供了两个hash相关的生命周期钩子：before-hash和after-hash。源码如下：

```
this.applyPlugins("before-hash");
this.createHash();
this.applyPlugins("after-hash");
```

hash可以作为版本控制的一环，将其作为编译输出文件夹的名称统一管理，如下：

```
output: {
	filename: '/dest/[hash]/[name].js'
}
```

### 8. chunkhash的问题

webpack的理念是一切都是模块：把所有类型的文件都以js为汇聚点，不支持js文件以外的文件为编译入口。所以如果我们要编译style文件，唯一的办法是在js文件中引入style文件。如下

```
import './style.css';

```

webpack默认将js、image、css文件统统编译到一个js文件中，

这样的模式下有个很严重的问题，当我们希望将css单独编译输出并且打上hash指纹，按照前文所述的使用chunkhash配置输出文件名时，编译的结果是js和css文件的hash指纹完全相同。


可以借助[extract-text-webpack-plugin](https://github.com/webpack-contrib/extract-text-webpack-plugin)将style文件单独编译输出。从这点可以看出，webpack将css文件视为js的一部分。


```
new ExtractTextPlugin('[name].[chunkhash].css');

```

但是不论是单独修改了js代码还是css代码，编译输出的js/css文件都会打上全新的相同的hash指纹。这种状况下我们无法有效的进行版本管理和部署上线。


好在extract-text-webpack-plugin提供了另外一种hash值：contenthash。顾名思义，contenthash代表的是文本文件内容的hash值，也就是只有style文件的hash值。这个hash值就是解决上述问题的银弹。修改配置如下：


```
new ExtractTextPlugin('[name].[contenthash].css');

```

编译输出的js和css文件将会有其独立的hash指纹。



再考虑一下这个问题：如果只修改了style文件，未修改index.js文件，那么编译输出的js文件的hash指纹会改变吗？

答案是肯定的。

修改了style编译输出的css文件hash指纹理所当然要更新，但是我们并未修改index.js，可是js文件的hash指纹也更新了。这是因为上文提到的：

webpack计算chunkhash时，以index.js文件为编译入口，整个chunk的内容会将style.css的内容也计算在内。


### 9. chunk-hash


webpack计算chunkhash时，以index.js文件为编译入口，整个chunk的内容会将style.css的内容也计算在内：

body{
    color: #000;
}
alert('I am main.js');

chunk-hash并不是webpack中另一种hash值，而是compilation执行生命周期中的一个钩子。

chunk-hash钩子代表的是哪个阶段呢？请看webpack的Compilation.js[https://github.com/webpack/webpack/blob/master/lib/Compilation.js#L1233]源码中以下部分：

```
for(let i = 0; i < chunks.length; i++) {
	const chunk = chunks[i];
	const chunkHash = crypto.createHash(hashFunction);
	if(outputOptions.hashSalt)
		chunkHash.update(outputOptions.hashSalt);
	chunk.updateHash(chunkHash);
	if(chunk.hasRuntime()) {
		this.mainTemplate.updateHashForChunk(chunkHash, chunk);
	} else {
		this.chunkTemplate.updateHashForChunk(chunkHash, chunk);
	}
	this.applyPlugins2("chunk-hash", chunk, chunkHash);
	chunk.hash = chunkHash.digest(hashDigest);
	hash.update(chunk.hash);
	chunk.renderedHash = chunk.hash.substr(0, hashDigestLength);
}
this.fullHash = hash.digest(hashDigest);
this.hash = this.fullHash.substr(0, hashDigestLength);
```

webpack使用NodeJS内置的crypto模块计算chunkhash，具体使用哪种算法与我们讨论的内容无关，我们只需要关注上述代码中this.applyPlugins("chunk-hash", chunk, chunkHash);的执行时机。

chunk-hash是在chunhash计算完毕之后执行的，这就意味着如果我们在chunk-hash钩子中可以用新的chunkhash替换已存在的值。如下伪代码：

```
compilation.plugin("chunk-hash", function(chunk, chunkHash) {
	var new_hash = md5(chunk);
   chunkHash.digest = function () {
		return new_hash;
    };
});

```

webpack之所以如果流行的原因之一就是拥有庞大的社区和不计其数的开发者们，实际上，我们遇到的问题已经有先驱者帮我们解决了。插件[webpack-md5-hash](https://github.com/erm0l0v/webpack-md5-hash)便是上述伪代码的具体实现，我们需要做的只是将这个插件加入到webpack的配置中：

```
// webpack.config.js

var WebpackMd5Hash = require('webpack-md5-hash');

module.exports = {
    // ...
    output: {
        //...
        chunkFilename: "[chunkhash].[id].chunk.js"
    },
    plugins: [
        new WebpackMd5Hash()
    ]
};
```

#### 10. 结语

静态资源的版本管理是前端工程化中非常重要的一环，使用webpack作为构建工具时需要谨慎使用hash和chunkhash，并且还需要注意webpack将一切视为js模块这种理念带来的一些不便。


