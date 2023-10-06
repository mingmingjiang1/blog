# 设计模式&渲染模式&优化&React

## Design Pattern

Singleton pattern：This *single instance* can be shared throughout our application, which makes Singletons great for managing global state in an application.

特点：

1. Singletons are classes which can be instantiated once
2. can be accessed globally

```
let instance;
let counter = 0;
 
class Counter {
  constructor() {
    if (instance) {
      throw new Error("You can only create one instance!");
    }
    instance = this;
  }
 
  getInstance() {
    return this;
  }
 
  getCount() {
    return counter;
  }
 
  increment() {
    return ++counter;
  }
 
  decrement() {
    return --counter;
  }
}
 
const singletonCounter = Object.freeze(new Counter());
export default singletonCounter;
```

## Tradeoffs

In many programming languages, such as Java or C++, it's not possible to directly create objects the way we can in JavaScript. In those object-oriented programming languages, we need to create a class, which creates an object. That created object has the value of the instance of the class, just like the value of `instance` in the JavaScript example.

However, the class implementation shown in the examples above is actually overkill. Since we can directly create objects in JavaScript, we can simply use a regular object to achieve the exact same result. Let's cover some of the disadvantages of using Singletons!



js的字面量很容易写出单粒模式

```
let count = 0;

const counter = {
  increment() {
    return ++count;
  },
  decrement() {
    return --count;
  }
};

Object.freeze(counter);
export { counter };
```



In React, we often rely on a global state through state management tools such as **Redux** or **React Context** instead of using Singletons. Although their global state behavior might seem similar to that of a Singleton, these tools provide a **read-only state** rather than the *mutable* state of the Singleton. When using Redux, only pure function *reducers* can update the state, after a component has sent an *action* through a *dispatcher*.

Although the downsides to having a global state don't magically disappear by using these tools, we can at least make sure that the global state is mutated the way we intend it, since components cannot update the state directly.





Proxy

代理对象控制了我们和原对象交互时的行为，它拦截了任何action which interact with object

```
const person = {
  name: "John Doe",
  age: 42,
  nationality: "American",
};
 
const personProxy = new Proxy(person, {});
```

https://res.cloudinary.com/ddxwdqwkr/video/upload/f_auto/v1609056520/patterns.dev/jspat-51_xvbob9.mp4







Provider 



```
const DataContext = React.createContext()
 
function App() {
  const data = { ... }
 
  return (
    <div>
      <DataContext.Provider value={data}>
        <SideBar />
        <Content />
      </DataContext.Provider>
    </div>
  )
}
```

https://res.cloudinary.com/ddxwdqwkr/video/upload/f_auto/v1609056518/patterns.dev/jspat-48_jxmuyy.mp4







Prototype







## Performance

### preload vs prefetch



### async vs defer

现代的网站中，脚本往往比 HTML 更“重”：它们的大小通常更大，处理时间也更长。

当浏览器加载 HTML 时遇到 `<script>...</script>` 标签，浏览器就不能继续构建 DOM。它必须立刻执行此脚本。对于外部脚本 `<script src="..."></script>` 也是一样的：浏览器必须等脚本下载完，并执行结束，之后才能继续处理剩余的页面。



这会导致两个重要的问题：

1. 脚本不能访问到位于它们下面的 DOM 元素，因此，脚本无法给它们添加处理程序等。
2. 如果页面顶部有一个笨重的脚本，它会“阻塞页面”。在该脚本下载并执行结束前，用户都不能看到页面内容



🌰：

```html
<p>...content before script...</p>

<script src="https://javascript.info/article/script-async-defer/long.js?speed=1"></script>

<!-- This isn't visible until the script loads -->
<p>...content after script...</p>
```



这里有一些解决办法。例如，我们可以把脚本放在页面底部。此时，它可以访问到它上面的元素，并且不会阻塞页面显示内容：

```html
<body>
  ...all content is above the script...

  <script src="https://javascript.info/article/script-async-defer/long.js?speed=1"></script>
</body>
```

但是这种解决方案远非完美。例如，浏览器只有在下载了完整的 HTML 文档之后才会下载该脚本（获取更多的资源）。对于长的 HTML 文档来说，这样可能会造成明显的延迟。

如下例：

```html
<body>
  ...all content is above the script...
	100000 lines omit ..........
  <script src="https://javascript.info/article/script-async-defer/long.js?speed=1"></script>
</body>
```

执行上面100000行，这时候遇到了script才去下载。有一种想法：script能不能放在前面，只是提前获取这个脚本，但是不执行，这样就不会，阻塞后续DOM的解析了。由此产生了defer：

defer：遇到脚本，先下载，但是不执行，可以想象在js单线程的场景下如何实现——队列，这些任务放在队列里[promise1, promise2]，呆到特定时机（DomContentLoaded）去执行这个队列，队列里任务的顺序和脚本出现的顺序相关，所以defer有一个优点，很好地通过手动编排script出现的顺序保证了存在依赖脚本的加载顺序。



可不可以通过并行的角度实现呢？

遇到脚本，下载的同时并执行，async就是这样的思路，每个附带async的script都是独立的一个任务，放在单独一个线程里去下下载并执行，但是这样不能保证脚本的执行顺序。



这对于使用高速连接的人来说，这不值一提，他们不会感受到这种延迟。但是这个世界上仍然有很多地区的人们所使用的网络速度很慢，并且使用的是远非完美的移动互联网连接。

幸运的是，这里有两个 `<script>` 特性（attribute）可以为我们解决这个问题：`defer` 和 `async`。

> 参考：
>
> https://zh.javascript.info/script-async-defer



https://zhuanlan.zhihu.com/p/48521680



缓存

### Server push

### http itself

### client

### Bundle splitting: Split your code into small, reusable pieces

### loading sequence

### performance metrics



### tree shaking

wip



### lazy loading

wip



### route Base Splitting

Dynamically load components based on the current route



### Dynamic import vs static import

1. Static import
2. Dynamic import: 仅导入你需要的模块

3. Load non-critical components when they are visible in the viewport
4. Load non-critical resources when a user interacts with UI requiring it



### list virtualization

wip



### compression: 

JavaScript is the second biggest [contributor to page size](https://almanac.httparchive.org/en/2020/page-weight#fig-2) and the second most [requested web resource](https://almanac.httparchive.org/en/2020/page-weight#fig-4) on the internet after images. We use patterns that reduce the transfer, load, and execution time for JavaScript to improve website performance. Compression can help reduce the time needed to transfer scripts over the network.

js是和页面大小相关的第二大重要因素(继图片之后)，压缩js可以减少传输时间

可以把压缩js和以下方法减少大js的影响：

- minification

- code-splitting

- bunding

- caching

- lazy-loading





#### HTTP compression

Compression reduces the size of documents and files, so they take up less disk space than the originals. Smaller documents consume lower bandwidth and can be transferred over a network quickly. HTTP compression uses this simple concept to compress website content, reduce [page weights](https://almanac.httparchive.org/en/2020/page-weight), lower bandwidth requirement, and improve performance.

HTTP data compression may be categorized in different ways. One of them is lossy vs. lossless.

**Lossy compression** implies that the compression-decompression cycle results in a slightly altered document while retaining its usability. The change is mostly imperceptible to the end-user. The most common example of lossy compression is JPEG compression for images.

With **Lossless compression,** the data recovered after compression and subsequent decompression will match precisely with the original. PNG images are an example of lossless compression. Lossless compression is relevant to text transfers and should be applied to text-based formats such as HTML, CSS, and JavaScript.

Since you want all valid JS code on the browser, you should use lossless compression algorithms for JavaScript code. Before we compress the JS, minification helps eliminate the unnecessary syntax and reduce it to only the code required for execution.

[HTTP 协议中的数据压缩 - HTTP | MDN (mozilla.org)](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Compression)



#### Minification

在压缩之前可以minify

 [Minification](https://web.dev/reduce-network-payloads-using-text-compression/#minification) complements compression by removing whitespace and any unnecessary code to create a smaller but perfectly valid code file. When writing code, we use line breaks, indentation, spaces, well-named variables, and comments to improve code readability and maintainability. 



Minification is a standard practice for JS and CSS optimization. It's common for JavaScript library developers to provide minified versions of their files for production deployments, usually denoted with a min.js name extension. (e.g., `jquery.js` and `jquery.min.js`)

Multiple tools are available for [the minification of HTML, CSS, and JS](https://developers.google.com/speed/docs/insights/MinifyResources) resources. [Terser](https://github.com/terser-js/terser) is a popular JavaScript compression tool for ES6+, and [Webpack](https://webpack.js.org/) v4 includes a plugin for this library by default to create minified build files. You can also use the `TerserWebpackPlugin` with older versions of Webpack or use Terser as a CLI tool without a module bundler.



#### Compression

服务侧一般有两种压缩方式：

1. static compression: 在项目构建的时候压缩，一般压缩不常变化的静态资源，可以较高程度压缩，虽然压缩时间比较长
2. dynamic compression: 当资源请求的时候才压缩，但是动态压缩一半压缩程度较低，因为压缩程度高，花费时间比较长，对于小型资源，也没啥优势，一般常用语动态资源。



#### 压缩算法：

1. Gzip

   The Gzip compression format has been around for almost 30 years and is a lossless algorithm based on the [Deflate algorithm](https://www.youtube.com/watch?v=whGwm0Lky2s&t=851s). The deflate algorithm itself uses a combination of the [LZ77 algorithm](https://cs.stanford.edu/people/eroberts/courses/soco/projects/data-compression/lossless/lz77/algorithm.htm) and [Huffman coding](https://cs.stanford.edu/people/eroberts/courses/soco/projects/data-compression/lossless/huffman/algorithm.htm) on blocks of data in an input data stream.

   The LZ77 algorithm identifies duplicate strings and replaces them with a backreference, which is a pointer to the place where it previously appeared, followed by the length of the string. Subsequently, Huffman coding identifies the commonly used references and replaces them with references with shorter bit sequences. Longer bit sequences are used to represent infrequently used references.

   All major browsers support Gzip. The [Zopfli](https://github.com/google/zopfli) compression algorithm is a slower but improved version of Deflate/Gzip, producing smaller GZip compatible files. It is most suitable for static compression, where it can provide more significant gains.

![img](https://www.patterns.dev/_next/image?url=%2Fimg%2Fcompression%2Fcompressingjav--zhfjmtap05.png&w=3840&q=75)

2. Brotli





#### Check Compression

Chrome -> DevTools -> network -> Headers. DevTools displays the content-encoding used in the response, as shown below.

![img](https://www.patterns.dev/_next/image?url=%2Fimg%2Fcompression%2Fcompressingjav--4gwntp0et8s.png&w=3840&q=75)



The lighthouse report includes a performance audit for "Enable Text Compression" that checks for text-based resource types received without the content-encoding header set to ‘br', ‘gzip' or ‘deflate'. Lighthouse uses Gzip to compute the potential savings for the resource.

![img](https://www.patterns.dev/_next/image?url=%2Fimg%2Fcompression%2Fcompressingjav--qmwdq1rskk8.png&w=3840&q=75)

#### trade off

1. gain(1+2) >= gain(1) + gain(2): 放在一起压缩的收益比分开压缩的收益更好

   Limited local data suggests a 5% to 10% loss for smaller chunks. The extreme case of unbundled chunks shows a 20% increase in size. Additional IPC, I/O, and processing costs are attached to each chunk that gets shared in the case of larger chunks. The v8 engine has a 30K streaming/parsing threshold. This means that all chunks smaller than 30K will parse on the critical loading path even if it is non-critical.

2. 但是分开压缩对缓存有好处：

   1. 如果某个资源发生改变，分开压缩的模块的情况下只需要重新获取对应的最新资源即可；但是如果是大文件压缩，则需要重新获取整个资源；
   2. 另外，按需加载的情况下，我们也是尽可能只需要尽量小的资源，如果整个都压缩在一起了，势必也会不好，下载了很多无用的部分

As a result of this trade-off, the maximum number of chunks used today by most production apps is around 10. This limit needs to be increased to support better caching and de-duplication for apps with large amounts of JavaScrip



## Render Patterns

UX friendly

<img src="https://www.patterns.dev/_next/image?url=https%3A%2F%2Fres.cloudinary.com%2Fddxwdqwkr%2Fimage%2Fupload%2Ff_auto%2Fv1660456914%2Fpatterns.dev%2Fweb-vitals.png&w=3840&q=75" alt="img" style="zoom: 25%;" />

DX friendly

![img](https://www.patterns.dev/_next/image?url=https%3A%2F%2Fres.cloudinary.com%2Fddxwdqwkr%2Fimage%2Fupload%2Ff_auto%2Fv1658990025%2Fpatterns.dev%2F5.png&w=3840&q=75)



trade off:

![img](https://www.patterns.dev/_next/image?url=https%3A%2F%2Fres.cloudinary.com%2Fddxwdqwkr%2Fimage%2Fupload%2Ff_auto%2Fv1658990025%2Fpatterns.dev%2F6.png&w=3840&q=75)

The Chrome team [has encouraged](https://developers.google.com/web/updates/2019/02/rendering-on-the-web) developers to consider static or server-side rendering over a full rehydration approach. Over time, progressive loading and rendering techniques, by default, may help strike a good balance of performance and feature delivery when using a modern framework.



### static Rendering

- 项目build的html就生成好了，直到下一次build之前都不会发生改变，所以页面比较死板
- 速度特别快
- cdn缓存策略加速用户访问
- 适合不常改变的幂等性网页(不论怎么请求结果都是一样的)
- 不存在re-layout and repainting

https://res.cloudinary.com/dq8xfyhu4/video/upload/l_logo_pke9dv,o_52,x_-1510,y_-900/ac_none/v1609691928/CS%20Visualized/Screen_Recording_2022-05-05_at_10.18.37_AM_bhybvb.webm



### static Rendering with client

- 改变了static rendering的非动态数据的特点，允许请求动态数据
- 同时为例避免re-layout and repainting，使用了骨架屏方案，防止数据渲染的时候引起UI变动；
- 但是骨架屏的大小要准确
- 给予了服务器压力，因为现在要实时获取数据了，每一个请求都需要server侧回应



https://res.cloudinary.com/dq8xfyhu4/video/upload/l_logo_pke9dv,o_52,x_-1510,y_-900/ac_none/v1609691928/CS%20Visualized/Screen_Recording_2022-05-05_at_2.55.30_PM_r0jvez.webm



### static with getStaticProps

- 在build时期就可以动态的去获取数据，减少了一部分在客户端的请求
- 但是这些请求与用户互关(not user-specific)
- 但是如果这样的请求过多的话，build时间很能会很长
- 这种方法也只适用于在构建时不经常更新数据的情况



https://res.cloudinary.com/dq8xfyhu4/video/upload/l_logo_pke9dv,o_52,x_-1510,y_-900/ac_none/v1609691928/CS%20Visualized/Screen_Recording_2022-05-05_at_3.06.26_PM_djvt57.webm





### Incremental sattic regeneration

对于上面build time的问题(static with getStaticProps)和动态数据(static Rendering with client)的问题，可以使用这个方法解决

- 和前面的思路不一样，之前是build 的时候一次性产生所有静态资源，而现在是请求一个html，就重新生成一个（有优先从缓存里取），一般配合serveless function一起使用（减少了build time），数据仍然是在server侧生成的
- 不需要build，但是似乎需要重新部署到cdn，为了避免重新部署，定时校验静态资源缓存
- Thus, only the first user is likely to have a poorer experience for pages that are not pre-rendered.

https://res.cloudinary.com/dq8xfyhu4/video/upload/l_logo_pke9dv,o_52,x_-1510,y_-900/ac_none/v1609691928/CS%20Visualized/Screen_Recording_2022-05-05_at_3.49.59_PM_deygni.webm

https://res.cloudinary.com/dq8xfyhu4/video/upload/l_logo_pke9dv,o_52,x_-1510,y_-900/ac_none/v1609691928/CS%20Visualized/updated_jvhqnv.webm



### On-demand Incremental sattic regeneration

- not user-specific

- 不是定时更新cdn，而是基于特定事件

  



### SSR

With server-side rendering, we generate the HTML for every request. This approach is most suitable for pages containing highly personalized data, for example, data based on the user cookie or generally any data obtained from the user's request. It's also suitable for pages that should be render-blocking, perhaps based on authentication state.

- 可以和csr一样包含了user-specific data
- use request-based data, like cookie
- should be render-blocking
- The time it takes to start up the lambda, known as the long cold boot, is a common issue with serverless functions. Also, connections to databases can be slow. You should also not call a serverless function located on one side of the planet from the other.
- server侧压力大了，优化方法：
  - **Deploy databases in the same region as your serverless function**：减少查询时间
  - **Execution time of `getServerSideProp`**：The page generation does not start until the data from `getServerSideProps` is available. Hence, we must ensure that the `getServerSideProps` method doesn't run too long.
  - **Add `Cache-control` headers to responses**
  - **Upgrade server hardware**



https://res.cloudinary.com/dq8xfyhu4/video/upload/l_logo_pke9dv,o_52,x_-1510,y_-900/ac_none/v1609691928/CS%20Visualized/Screen_Recording_2022-05-05_at_5.31.41_PM_oxsq12.webm



### CSR

<模板 js执行的逻辑会挂在这个节点下面>

<script> 如果这的逻辑很长或者下载该脚本及其依赖脚本时间过长，就会导致白屏



SSR: 

实际上SSR的一定有的结构如下：

```
<div id='root'>
	<>{renderToHtml(content)}<>
	<script src='client.js' />
</div>
```

把server侧的整个html交给了客户侧的js管理，但是这样每次请求一个新页面，就会重新刷新一次，它的粒度不够细，RSC可以解决这个问题。



关于SSR的实践的几个点：

- 同构是什么？server无法处理的一些事情，如事件绑定交给客户端
- 客户端和server侧同步store，为什么要同步呢？
- 路由的问题：如果要使用局部刷新的话，ssr要写两份路由，一份是客户端的路由，一份是对应的服务端路由



SSR的实践：

[React SSR 初实践（一） - 掘金 (juejin.cn)](https://juejin.cn/post/7065303971723739144)



### React Server Component

一句话来说：RSC可以使得客户端的React树既有客户端组建也有服务端组件，所以整个React树是可以实现局部刷新的

[React Server Components到底行不行？ - 掘金 (juejin.cn)](https://juejin.cn/post/6931902782131666951#heading-7)

[React server component 深入指南 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/470449193)



### HTTP streaming

[普通的请求结果：0 or 1，stream：随时使用，挤牙膏]

With Edge SSR, we can stream parts of the document as soon as they're ready and hydrate these components granularly. This reduces the waiting time for users as they can see components as they stream in one by one.

https://res.cloudinary.com/dq8xfyhu4/video/upload/l_logo_pke9dv,o_52,x_-1510,y_-900/ac_none/v1609691928/CS%20Visualized/Screen_Recording_2022-05-05_at_5.48.20_PM_auurip.webm



结合：静态数据网页static rendering，user-specific server rendering

[从 Fetch 到 Streams —— 以流的角度处理网络请求 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/98848420)



### Some ideas in Reacting

### 关于React性能优化







### 关于写组件的一些体会

headless UI: 无UI组件，表示仅提供UI元素和交互的数据状态逻辑，但不提供标记(html元素)，样式



暴露状态和一些控制逻辑给用户？

但是状态在内部，用户不需要管，以及控制该状态的api也已经给用户了，具体如何使用该api控制该状态，进而控制UI全权交给了用户



https://www.patterns.dev/posts/hoc-pattern