# refreshToken
axios 接口请求时token过期，自动刷新token
1.在开发过程中，我们都会接触到token，token的作用是什么呢？主要的作用就是为了安全，用户登陆时，服务器会随机生成一个有时效性的token,用户的每一次请求都需要携带上token，证明其请求的合法性，服务器会验证token，只有通过验证才会返回请求结果。

2.当token失效时，现在的网站一般会做两种处理，一种是跳转到登陆页面让用户重新登陆获取新的token，另外一种就是当检测到请求失效时，网站自动去请求新的token，第二种方式在app保持登陆状态上面用得比较多。

3.下面进入主题，我们请求用的是axios，不管用何种请求方式，刷新token的原理都是一样的。

//封装了一个统一的请求函数，这个不是重点
```
export default function request(url, options) {
    const token = localStorage.getItem('token');
    const defaultOptions = {
        headers: {
            Authorization: `Bearer ${token}`,
        },
        withCredentials: true,
        url: url,
        baseURL: BASE_URL,
    };
    const newOptions = { ...options, ...defaultOptions };
    return axios.request(newOptions)
        .then(checkStatus)
        .catch(error => console.log(error));
}
```
// 封装了一个检测返回结果的函数，与后台返回状态码code === 1002表示token失效

```
let isRefreshing = true;
function checkStatus(response) {
  if (response && response.code === 1002) {
    // 刷新token的函数,这需要添加一个开关，防止重复请求
    if(isRefreshing){
        refreshTokenRequst()
    }
    isRefreshing = false;
    // 这个Promise函数很关键
      const retryOriginalRequest = new Promise((resolve) => {
                addSubscriber(()=> {
                    resolve(request(url, options))
                })
            });
            return retryOriginalRequest;
  }else{
      return response;
  }
}
```
// 刷新token的请求函数
```
function refreshTokenRequst(){
    let data;
    const refreshToken = localStorage.getItem('refreshToken');
    data:{
        authorization: 'YXBwYXBpczpaSWxhQUVJdsferTeweERmR1praHk=',
        refreshToken,
    }
    axios.request({
        baseURL: BASE_URL,
        url:'/app/renewal',
        method: 'POST',
        data,
    }).then((response)=>{
        localStorage.setItem('refreshToken',response.data.refreshToken);
        localStorage.setItem('token',response.data.token);
        onAccessTokenFetched();
        isRefreshing = true;
    });
}
```
// Promise函数集合 
```
let subscribers = [];
function onAccessTokenFetched() {
    subscribers.forEach((callback)=>{
        callback();
    })
    subscribers = [];
}

function addSubscriber(callback) {
    subscribers.push(callback)
}
```
总结：其实token失效，自动刷新token，在页面只有一个请求的时候是比较好处理的，但是如果页面同时有多个请求，并且都会产生token失效，这就需要一些稍微复杂的处理，解决方式主要是用了Promise 函数来进行处理。每一个token失效的请求都会存到一个Promise函数集合里面，当刷新token的函数执行完毕后，才会批量执行这些Promise函数，返回请求结果。还有一点要注意一下，这儿设置一个刷新token的开关isRefreshing，这个是非常有必要的，防止重复请求。
