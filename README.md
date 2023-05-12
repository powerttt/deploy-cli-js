# deploy-cli-js

快速部署到海外服务器。将部署文件上传到 `S3` 再通过跳板机前往对应的机器执行部署

# 使用场景

海外的 VPS 延迟较高，部署不是很流畅，国内可是使用 [deploy-cli-service ](https://github.com/fuchengwei/deploy-cli-service)
脚本使用 node 编写，代码全部来自于 ChatGPT。方法就是通过一台延迟比较低的服务器 ssh 连上后再连另一台延迟较高的服务器，再执行对应的命令，有其他需要可以自行更改，DIY 编写的灵活度较高。

下面的示例是一个 vue 项目，先将 dist 打包成 zip，上传到 s3，连接 ssh，下载到指定目录再解压。

# 步骤

## 依赖

在原有的 node 或 js 项目上安装依赖到 dev
- 若是其他项目则要重新初始化一下 `npm init `
```
pnpm i fs archiver aws-sdk ssh2 dayjs -D
```
- 使用到 `aws-sdk` 需要去看下对应的文档初始化一下本地的 aws 密钥

## 创建 `device.upload.js`

```js

const fs = require('fs');
const archiver = require('archiver');
const AWS = require('aws-sdk');
const Client = require('ssh2').Client;
const dayjs = require('dayjs')
  
const jumpHost = {
    host: 'xxx.xxx.xxx.xxx',
    port: 22,
    username: 'root',
    password: 'xxxxxx'

};
const targetServer = {

    host: 'xxx.xxx.xxx.xxx',
    port: 22,
    username: 'root',
    password: 'xxxxxx'

};

const today = dayjs().format('MMDD_HHmm')
const filename = `dist_${today}.zip`
const fileS3Key = `public/xxxxxx/${filename}`
const fileSite = `https://xxxxxx.s3.amazonaws.com`

const fileUrl = `${fileSite}/${fileS3Key}`
const webDir = "/www/wwwroot/xxx.xxx.com"
const backDir = "/opt/works/xxx/xxx_back"
const fileS3Bucket = "xxx"
async function zip () {
    const output = fs.createWriteStream(filename);
    const archive = archiver('zip', {
        zlib: { level: 9 } // Sets the compression level.
    });

    output.on('close', function () {
        console.log(archive.pointer() + ' total bytes');
        console.log('archiver has been finalized and the output file descriptor has closed.');
    });

    archive.on('error', function (err) {
        throw err;
    });

    archive.pipe(output);

    // 从子目录追加文件并在存档中将其命名为“new-subdir”（您也可以像这样将文件调用链接在一起）

    // 如果为 destpath 传递 false，则存档中数据块的路径设置为根目录

    archive.directory('./dist', false);

    return archive.finalize();
}


async function uploadS3 () {
    return new Promise((resolve, reject) => {
        const s3 = new AWS.S3();

        const fileStream = fs.createReadStream(filename);

        let uploadedBytes = 0;

        fileStream.on('data', function (chunk) {

            uploadedBytes += chunk.length;

            console.log('Uploaded', uploadedBytes / 1024, 'bytes');

        });

  

        const uploadParams = {
            Bucket: fileS3Bucket,
            Key: fileS3Key,
            Body: fileStream,
            partSize: 2 * 1024 * 1024, // 10 MB per part
            queueSize: 10 // 10 parts uploaded in parallel
        };

        s3.upload(uploadParams, function (err, data) {
            if (err) {
                console.log('Error uploading file:', err);
                reject(err)
            } else {
                console.log('File uploaded successfully:', data.Location);
                resolve(data)
            }
        });
    })
}

  

async function sshServer () {
    return new Promise((resolve, reject) => {
        const jumpConnection = new Client();
        jumpConnection.on('ready', () => {
            jumpConnection.forwardOut(
                '127.0.0.1',
                2222,
                targetServer.host,
                targetServer.port,
                (err, stream) => {
                    if (err) throw err;
                    const targetConnection = new Client();
                    targetConnection.on('ready', () => {
                        //  wget -c  -P /opt/works/xxx/xxx_back "https://xxx/dist_0426_1444.zip" && unzip -d /root/temp/0426 dist_0426_1444.zip

                        let exec = `wget -c  -P ${backDir} "${fileUrl}" && unzip -o -d ${webDir} ${backDir}/${filename}`

                        console.log(`exec ${exec}`)

                        targetConnection.exec(exec, (err, stream) => {

                            if (err) {

                                reject(err)

                                throw err

                            }

                            stream.on('close', (code, signal) => {

                                console.log(`Command exited with code ${code}`);

                                targetConnection.end();

                                jumpConnection.end();

                                resolve()

                            }).on('data', (data) => {

                                console.log(`STDOUT: ${data}`);

                            }).stderr.on('data', (data) => {

                                console.log(`STDERR: ${data}`);

                            });

                        });

                    }).connect({
                        sock: stream,
                        username: targetServer.username,
                        password: targetServer.password
                    });
                }
            );
        }).connect(jumpHost);
    })
}

  

let startTime = new Date().getTime()
zip()
    .then(() => uploadS3())
    .then(() => sshServer())
    .then(() => {
      console.log(`All promises executed\n use time ${new Date.getTime() - startTime }ms`)
    });
```

## 执行脚本

```shell
node deploy.upload.js
```

或者也可以写在 `package.json`，在原本的命令上加上 `&& node deploy.upload.js`

```shell
 "deploy": "vite build --mode prod && node deploy.upload.js"
```
- `.gitignore` 添加  `deploy.*.js dist*.zip`

# 扩展

- [ ] 将 jar 上传并改写前置后置命令

# 说明

- 只是一个示例，也可以把 s3 相关的步骤给去掉，换成 `scp` 代替
- 多环境的花也别封装了，再复制一份改个名字改参数比较合理
- 代码全部来自 ChatGPT
