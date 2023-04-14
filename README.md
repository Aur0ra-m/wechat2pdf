# wechat2pdf
> 一键下载微信公众号文章PDF版

## 简介
> 基于https://www.wechat2pdf.com/的油猴插件脚本

**打开公众号文章后，左上角有个下载按钮，点击即可**
![image](https://user-images.githubusercontent.com/103031059/232041004-b663e706-c0b8-4b16-bda3-bef793514e4f.png)

![image](https://user-images.githubusercontent.com/103031059/232041340-2eeb49d4-2a90-447c-bef5-310d03bb5509.png)


## 油猴脚本安装
> 在油猴插件中新建脚本，将下面的代码复制进去即可

```js
// ==UserScript==
// @name         wechat2pdf download plugin
// @namespace    https://www.example.com/
// @version      1.0
// @description  download wechat article in form of PDF
// @author       Aur0ra
// @match        https://mp.weixin.qq.com/*
// @run-at       document-end
// @grant        GM_xmlhttpRequest
// ==/UserScript==

(function() {
    'use strict';

    function downloadPDF(){
        // 更新状态
        const button = document.getElementById('download-pdf-button');
        if (!button) {
            return;
        }
        button.disabled = true;
        button.innerText = 'Downloading...';

        // 获取当前页面的URL
        const url = window.location.href;
        console.log("获取资源："+url)

        // 发送HTTP POST请求以将URL提交到https://www.wechat2pdf.com/apis/conversions
        GM_xmlhttpRequest({
            method: 'POST',
            url: 'https://www.wechat2pdf.com/apis/conversions',
            headers: {
                'Content-Type': 'application/json',
                'Origin': 'https://www.wechat2pdf.com'  // 跨域请求头
            },
            data: JSON.stringify({showimg: true, url: url}),
            onload: response => {
                const data = JSON.parse(response.responseText);
                // 解析返回的JSON响应以查找poll_url
                if (data && data.poll_url) {
                    console.log(`已提交URL转PDF请求，poll_url为${data.poll_url}`);
                    setTimeout( function(){
                        //add your code
                    }, 6 * 1000 );//延迟6000毫秒
                    // 开始轮询获取下载链接
                    let retries = 0;
                    const maxRetries = 600;  // 默认10分钟超时（每秒一次轮询）
                    const intervalId = setInterval(() => {
                        GM_xmlhttpRequest({
                            method: 'GET',
                            url: `https://www.wechat2pdf.com${data.poll_url}`,
                            headers: {'Origin': 'https://www.wechat2pdf.com'},
                            onload: response => {
                                const data = JSON.parse(response.responseText);
                                if (data && data.state) {
                                    if (data.state === 'PENDING' && retries < maxRetries) {
                                        console.log(`PDF生成中（第${retries + 1}次轮询）...`);
                                        button.innerText = 'Downloading...['+ (retries + 1) +'s]';
                                        retries++;
                                    } else if (data.state === 'SUCCESS') {
                                        clearInterval(intervalId);
                                        if (data.download_url) {
                                            // 复制下载链接到剪贴板
                                            const downloadUrl = `https://www.wechat2pdf.com${data.download_url}`;
                                            const tempInput = document.createElement('input');
                                            tempInput.value = downloadUrl;
                                            document.body.appendChild(tempInput);
                                            tempInput.select();
                                            document.execCommand('copy');
                                            document.body.removeChild(tempInput);
                                            console.log(`PDF生成完成，已复制下载链接：${downloadUrl}`);
                                            // 下载文件
                                            window.open(downloadUrl, '_blank');
                                            button.innerText = 'Finished';
                                        } else {
                                            console.log('未找到PDF下载链接');
                                        }
                                    } else {
                                        clearInterval(intervalId);
                                        console.log(`PDF生成失败，状态为${data.state}`);
                                    }
                                } else {
                                    clearInterval(intervalId);
                                    console.log('无效的响应数据');
                                    // 复原状态
                                    button.disabled = false;
                                    button.innerText = 'Download PDF';
                                }
                            },
                            onerror: error => {
                                clearInterval(intervalId);
                                console.error(`发生错误：${error}`);
                                // 复原状态
                                button.disabled = false;
                                button.innerText = 'Download PDF';
                            }
                        });
                    }, 1000);  // 每秒一次轮询
                } else {
                    console.log('未找到poll_url');
                }
            },
            onerror: error => {
                console.error(`发生错误：${error}`);
            }
        });

    };

    function createDownloadButton() {
        const button = document.createElement('button');
        button.innerText = 'Download PDF';
        button.id = "download-pdf-button";
        button.style.cssText = `
            position: fixed;
            top: 20px;
            left: 20px;
            z-index: 9999;
            padding: 10px;
            background-color: #007fff;
            color: #fff;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        `;
        button.addEventListener('click', downloadPDF);
        document.body.appendChild(button);
    }

    createDownloadButton();
})();


```
