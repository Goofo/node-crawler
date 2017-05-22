#Node爬虫学习

知识点:
ES6标准Buffer使用.

ES7标准Async/Await and Promise使用.

Node child_process 简单使用.

Npm模块 cheerio,log4js使用.

Usage:

        npm i
        node meizitu.js

刚接触爬虫这种业务,对jquery也不熟悉,但是真简单,网上找了个教程就开始了.

计划这个Repository,以后慢慢更新爬虫相关的学习代码.

>抓取妹子图 http://www.meizitu.com 所有图片

核心函数有三个

1:获取网页HTML,使用cheerio模块解析DOM.

    /// 获取指定的URL的HTML文本
    function fetch(url) {
        return new Promise((resolve, reject)=> {
            logger.debug("async fetch url %s", url);
            http.get(url, response=> {
                let b = new Date();
                /// 其实nodejs默认的就是utf8,注意网站使用的GBK,所以中文是乱码.
                response.setEncoding('utf8');
                /// 访问出错,直接结束异步promise
                response.on('error', reject);
                let message = [];
                response.on('data', data=> {
                    message.push(Buffer.from(data));
                });
                let timer = setInterval(()=> {
                    logger.warn("fetch url %s running[%d]", url, new Date() - b);
                }, 1 * 1000 * 60);

                response.on('end', ()=> {
                    logger.debug("fetch url %s completed!", url);
                    let html = Buffer.concat(message);
                    clearInterval(timer);
                    resolve(cheerio.load(html.toString()));
                });
            })
        });
    }

2:下载图片,并发下载图片.

    /// 下载图片
    function fetchImage(source) {
        return new Promise((resolve, reject)=> {
            logger.debug("will download image %s", source);
            http.get(source, response=> {
                let now = new Date();
                /// 下载数据,这里需要使用binary,如果使用utf8还需要自己转换.
                response.setEncoding('binary');
                response.on('error', error=> {
                    logger.error("fetch image %s has error: %s", source, error);
                    reject(error);
                });
                let img = [];
                response.on('data', chunk=> {
                    //logger.debug("receive message chunk(%d) from %s", chunk.length, source);
                    img.push(Buffer.from(chunk, 'binary'));
                });
                let timer = setInterval(()=> {
                    /// 下载失败的图片 断开连接
                    response.statusCode = -1;
                    response.emit('end');
                    logger.warn("fetch url %s failed[%d]", source, new Date() - now);
                }, 2 * 60000);
                response.on('end', ()=> {
                    clearInterval(timer);
                    if (response.statusCode == 200) {
                        let data = Buffer.concat(img);
                        let name = crypto.createHash('sha256').update(data.toString()).digest('hex');
                        let dir = path.join(__dirname, 'downloads', name + path.extname(source));
                        if (fs.existsSync(dir)) {
                            logger.warn("file exits %s", source);
                            resolve();
                        } else {
                            fs.writeFile(dir, data, 'binary', error=> {
                                //logger.debug("save image size(%d) to %s", data.length, dir);
                                resolve(error);
                            });
                        }
                    } else {
                        logger.warn("down load image %s response status code: %d", source, response.statusCode);
                        resolve(response.statusCode);
                    }
                    logger.debug("download image %s completed[%d]", source, Date.now() - now);
                });
            });
        });
    }


3:启动工作子进程抓取下一页内容.

    /// 启动工作进程(这里是启动一个进程去处理一个相册页面)
    function startWorker(content) {
        return new Promise((resolve, reject)=> {
            if (!content || content == '') {
                resolve();
                return;
            }
            let child = cp.spawn(process.execPath, [require.main.filename, content]);
            logger.info("create child process %d working for %s", child.pid, content);
            child.stderr.on('data', function (chunk) {
                reject(chunk);
                process.stderr.write(chunk.toString());
            });
            child.stdout.on('data', function (chunk) {
                process.stdout.write(chunk.toString());
            });
            child.on('close', code=> {
                logger.info("child process %d exit by %d", child.pid, code);
                resolve(code);
            });
        });
    }

4:详细些一下ES7 Async

    /// 对相册页面做DOM解析
    async function fetchPage() {
        let $ = await fetch(url);

        /// 下载工作进程进行并发下载该相册内容
        let images = $('#maincontent #picture p img').toArray();
        for (let i in images) {
            /// 这里使用异步,每个worker进程,并发下载当前相册的所有图片.
            fetchImage($(images[i]).attr('src'));
        }

        /// 查看是否是页面内容,如果是页面内容,同步开启进程下载该相册内的数据.
        let albums = $('.wp-item .con .pic a').toArray();
        for (let i in albums) {
            await startWorker($(albums[i]).attr("href"));
        }

        /// 启动进程下载下一页.
        let currentPage = Number.parseInt($('.thisclass').text());
        if (currentPage == 1) {
            let pages = $('#wp_page_numbers ul li a').toArray();
            for (let i in pages) {
                let nextUrl = $(pages[i]).attr('href');
                if (nextUrl != "") {
                    if (/^\/a/.test(nextUrl)) {
                        nextUrl = defUrl + nextUrl;
                    } else {
                        nextUrl = defUrl + "/a/" + nextUrl;
                    }
                    await startWorker(nextUrl);
                }
            }
        }
    }

5: 总结

之前没搞过爬虫,对jquery也不了解,但是cheerio这个模块很简单的就能上手

es7 async 特性真是好东西, 让代码直观的看出那里异步, 方便的写同步.