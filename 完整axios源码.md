# 完整源码

首先`json-server  --delay 1000 db.json   `   

```html
<body>
    <button class="send" onclick="send()">发送</button>
    <button class="cancel" onclick="cancelSend()">取消</button>
    <script>
        //构造函数，接收用户传入的config，然后生成请求拦截器和响应拦截器
        function Axios(config) {
            this.config = config;
            this.interceptors = {
                request: new InterceptorManager(), // 请求拦截器
                response: new InterceptorManager() // 响应拦截器
            };
        }
        //发送请求的函数，无论是axios()，还是axios.get() axios.post()，都是指向的这个函数
        Axios.prototype.request = function (config) {
            //创建一个promise，现在一定是成功的
            let promise = Promise.resolve(config);
            // 队列，用于存放拦截器和实际请求 undefined用于占位
            const chains = [dispatchRequest, undefined];
            //遍历请求拦截器和响应拦截器，然后将请求拦截器放在队列前面，响应拦截器放在队列最后面
            //这样就能实现先执行请求拦截器，然后发送请求，然后执行响应拦截器，最后返回结果。
            this.interceptors.request.handlers.forEach(item => chains.unshift(item.fulfilled, item.rejected));
            this.interceptors.response.handlers.forEach(item => chains.push(item.fulfilled, item.rejected));
            //循环执行队列
            while (chains.length) {
                promise = promise.then(chains.shift(), chains.shift());
            }
            // 返回结果
            return promise;
        }
        // get方法 axios.get()
        Axios.prototype.get = function (config) {
            config.method = 'get';
            return this.request(config)
        }
        // post方法 axios.post()
        Axios.prototype.post = function (config) {
            config.method = 'post';
            return this.request(config)
        }
        //拦截器的构造函数，handlers存放一组回调函数，成功的和失败的
        function InterceptorManager() {
            this.handlers = [];
        }
        // 原型上的方法，创建拦截器的时候使用该方法，将回调存入handlers中，然后在request中取出放入chain队列
        InterceptorManager.prototype.use = function (fulfilled, rejected) {
            this.handlers.push({
                fulfilled,
                rejected
            })
        }
        //取消请求的构造函数，通过promise，控制是否取消，
        function CancelToken(executor) {
            let resolvePromise = null; //改变promise的状态为成功
            this.promise = new Promise(resolve => resolvePromise = resolve);
            // 向外暴露一个方法，该方法中是resolve()
            //这样就用户执行这个方法时，就会调用resolve()，this.promise成功
            executor(() => {
                resolvePromise();
            })
        }
        // 调用请求 浏览器环境是xhr，node环境是http，这里就以xhr为例
        function dispatchRequest(config) {
            //返回xhr的执行结果
            return xhrAdapter(config).then(
                res => {
                    // 获取请求成功的结果，然后处理，返回，处理这里省略了
                    return res
                },
                err => {
                    // 获取请求失败的结果，然后处理，返回，处理这里省略了
                    return new Error(err)
                }
            )
        }
        //真正发起请求的函数，使用xhr
        function xhrAdapter(config) {
            //返回一个promise对象
            return new Promise((resolve, reject) => {
                // 发送请求
                const xhr = new XMLHttpRequest();
                xhr.open(config.method, config.url); //初始化
                xhr.send(); //发送
                //绑定事件
                xhr.onreadystatechange = function () {
                    if (xhr.readyState === 4) {
                        //判断成功的条件
                        if (xhr.status >= 200 && xhr.status <= 300) {
                            //状态改为成功，然后将请求结果返回
                            resolve({
                                data: JSON.parse(xhr.response),
                                status: xhr.status,
                                statusText: xhr.statusText
                            });
                        }
                    }
                }
                // 判断传入的config是否有cancelToken属性，如果有
                // 当cancelToken.promise的状态成功时，执行xhr.abort()，取消请求
                if (config.cancelToken) {
                    config.cancelToken.promise.then(() => {
                        xhr.abort();
                        reject(new Error('请求已取消'));
                    })
                }
            })
        }

        //存放改变cancelToken.promise的函数
        let cancel = null;
        // 创建一个axios
        function createInstance() {
            //新建一个实例
            const context = new Axios({});
            // 将request方法绑定在instance，这时候就可以通过instance()发起请求了
            const instance = Axios.prototype.request.bind(context);
            // 将get()，post()方法添加到instance上，这时候可以通过instance.get()发起请求了。
            Object.keys(Axios.prototype).forEach(key => {
                instance[key] = Axios.prototype[key].bind(context)
            })
            //将config和interceptors添加到instance上，就可以访问instance.interceptors了
            Object.keys(context).forEach(key => {
                instance[key] = context[key]
            })
            return instance;
        }
        function send(){
            //每次点击发送时，重新新建一个axios
            const axios = createInstance();
            //设置请求拦截器
            axios.interceptors.request.use(
                config => {
                    // 做点什么
                    console.log('请求拦截器响应');
                    return config
                },
                err => {
                    console.log('请求拦截器失败');
                    Promise.reject(err)
                },
            )
            // 设置响应拦截器
            axios.interceptors.response.use(
                config => {
                    // 做点什么
                    console.log('响应拦截器响应');
                    return config
                },
                err => {
                    console.log('响应拦截器失败');
                    Promise.reject(err)
                },
            )
            //新建一个CancelToken实例，然后将改变cancelToken.promise的函数暴露出来，由cancel接收
            //这样cancel()就直接改变了cancelToken.promise的状态为成功，然后在request中取消请求
            const cancelToken = new CancelToken(c => {
                cancel = c;
            });
            axios({
                method: 'get',
                url: 'http://localhost:3000/posts',
                cancelToken,
            }).then(res => console.log(res));
        }
        //取消发送
        function cancelSend(){
            if (cancel !== null) {
                cancel();//改变cancelToken.promise状态
                cancel = null;//删除cancel
            }
        }
    </script>
</body>
```

