# 我的 API 请求演进史

最近在重写 CRUD 项目，发现了一些以前没有想到的东西。在这里做下总结。╰(\*°▽°\*)╯

## 远古时期

在我刚刚接触 JavaScript 的时候，我是这样请求 HTTP API 的：

```js
$("button").click(function () {
  $.ajax({
    url: "/demo.txt",
    success: function (result) {
      $("#div1").html(result);
    },
  });
});
```

jQuery 当时可是如日中天，它极大地简化了 JavaScript 编程，广受前端开发工程师喜爱。当时的我并没有深究 `$.ajax` 背后到底发生了什么，直到我后面选择了前端作为自己的主攻方向...

我现在知道了 `$.ajax` 其实是对 ActiveXObject 和 XMLHttpRequest 的封装。ActiveXObject 过于陈旧了，远古时期的东西... 那个 Internet Explorer 还逍遥自在的时期，我现在想都不敢想。

我曾简单地试用了一下原始的 XHR：

```js
var xhr = new XMLHttpRequest();
xhr.onreadystatechange = function () {
  if (xhr.readyState == 4 && xhr.status == 200) {
    document.getElementById("div1").innerHTML = xhr.responseText;
  }
};
xhr.open("GET", "/demo.txt");
xhr.send();
```

然后我放弃了直接使用它... 直接使用 XHR 感觉就像是在开手动挡的车，我获得掌控，我失去便利。

后面我自己封装了 XHR，这样我就只需要提供必要的请求参数和回调了。看起来挺美好的，然后我发现了更美好的实现：

## Promise-based

在 JavaScript 世界中，发生了一件大事，ECMAScript 2015 正式接受了 Promise。

Promise 倒也并不新鲜，jQuery 1.5 就有一个类似的叫 Deferred Object 的玩意儿，异步编程的老祖宗们也纷纷表示像什么 future、promise、delay、deferred 这样那样的，早定义了。

Promise 被广泛支持之后，Axios 这样的 Promise based HTTP client 就自然成为了前端请求 API 的首选。看看下面这个例子，哇，真的很爽，jQuery 什么的靠边站：

```js
axios
  .get("/user?ID=12345")
  .then(function (response) {
    // handle success
    console.log(response);
  })
  .catch(function (error) {
    // handle error
    console.log(error);
  })
  .finally(function () {
    // always executed
  });
```

## 并没有用过的 Fetch

Fetch API 算是一个很新的东西，看起来就像是 W3C 心目中的 Axios：

```js
const image = document.querySelector(".my-image");
fetch("flowers.jpg")
  .then(function (response) {
    return response.blob();
  })
  .then(function (blob) {
    const objectURL = URL.createObjectURL(blob);
    image.src = objectURL;
  });
```

怎么说呢... 我知道 Fetch 的时候已经用上 Axios 了，然后 Axios 因为基于 XHR，可以 abort()，可以有 ProgressEvent，反观 Fetch 这边，好像就只能等请求结束返回 Promise 对象，其他的啥也干不了，于是尝鲜之后继续抱着 Axios 写项目。

## TypeScript

我是最近才开始使用 TypeScript 的，新事物总得有个熟悉的过程嘛，然后随着我对 TypeScript 理解的加深，我的 Axios 用法也在发生一些细微的改变。

最开始的 JSDoc 版 Axios：

```js
// 这里省略一些无关紧要的登录态检测和请求去重代码

/**
 * 请求构造工厂
 * @param {import('axios').AxiosRequestConfig} config
 * @returns
 */
export const ARFactory = async (config) => Axios(config);
```

我当时都用上 JSDoc 了，而且还不厌其烦地反复写 JSDoc 注释... 我为什么不早点换 TypeScript 呢。

后面的 TypeScript 版 Axios：

```ts
export const ARFactory = (config: AxiosRequestConfig) => {
  return Axios(config).then((resp) => {
    return resp.data;
  });
};
```

直接写类型声明就是比 JSDoc 舒服！

然后进化到了泛型版类型声明：

```ts
export const RequestFactory = async <T>(
  config: AxiosRequestConfig
): Promise<ResponseWith<T>> => {
  return Axios(config).then((res) => {
    return res.data;
  });
};
```

用上泛型之后终于不用在 API 接口定义那里写反反复复的 `as Promise<blabla>` 了：

```ts
// Past
export const login = (loginInfo: {
  userName: string;
  password: string;
  verifyCode: string;
}) => {
  return RequestFactory({
    url: `${API_BASE_URL}/user/web/login`,
    method: "POST",
    data: loginInfo,
  }) as Promise<ResponseWith<string>>;
};

// Now
export const login = (loginInfo: {
  userName: string;
  password: string;
  verifyCode: string;
}) => {
  return RequestFactory<string>({
    url: `${API_BASE_URL}/user/web/login`,
    method: "POST",
    data: loginInfo,
  }); // 这里的 as 语句不再需要啦
};
```

## 可观测性

在一些比较随意的项目里面，我一般会这样处理接口请求出问题的情况：

```ts
return Axios(config).then(
  (res) => {
    return res.data;
  },
  (reason) => {
    // 如果接口请求出错，一律视作网络异常
    console.log(reason);
    return {
      message: "网络异常，请稍后重试",
      succeed: false,
    };
  }
);
```

反正只要 succeed 置为 false，页面就会知道请求寄寄了，至于是要重发请求呢，还是把问题抛给用户，那就不需要在这里考虑了。但在现实一点的项目里面，这样做大概率是要被婊起来，挂在墙上，遗臭万年的。

这里我进行了 console.log，也就 Debug 的时候可能会看一眼。真要项目上线了指不定出什么问题，然后用户就看这不断弹出的“网络异常”，看着自己还算正常的网络陷入沉思，与此同时，我还不知道这个情况。那... 最后，用户跑掉了，效益没了，我也该被优化了。

然后我就把这段代码改成了类似这样的：

```ts
return Axios(config).then(
  (res) => {
    return res.data;
  },
  (reason) => {
    logPlatform.reportAxiosError(reason); // 多了这玩意儿，用来上报数据

    return {
      message: reason.code, // message 换成 AxiosError 里面的 code 了，反正用户只要看到爆红就知道不对劲了...
      succeed: false,
    };
  }
);
```

然后如果接口因为某些原因没法正常请求了，就会有一个请求尽可能地飞向日志服务器，里面记载有请求的主要参数（URL、method、headers 这些），然后还可能有 response 的内容（服务器非 200 返回）。假如不是因为真的断网了啥的，就能通过接口出错的上报数据发现线上系统存在的问题了。

## END

哇... 打了好多字，我果然是大水比，睡觉了...
