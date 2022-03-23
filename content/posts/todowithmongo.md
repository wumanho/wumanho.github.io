---
weight: 2
title: "【Node.js】结合 MongoDB 实现一个 TODO 命令行工具"
summary: "Node.js + MongoDB 练手，做个 TODO 小工具"
date: 2021-07-11T15:54:49+08:00
draft: false
categories: ["技术"]
tags: ["JavaScript","Node.js","MongoDB"]
---

## 目标

本篇博客的目标是使用 Node.js 实现一个基于命令行的 TODO 小工具，数据将会被保存在 MongoDB 数据库中，包含基本的增删改查功能，最终效果大概如下：

![功能](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/nodemongo/use.gif)

虽然功能比较简陋，但用来练习应该是个不错的选择。

实现功能主要涉及的技术及第三方库包括：

- Node.js(javaScript 运行时)
- MongoDB(数据库)
- [Mongoose(node 基于对象模型的 mongo 库)](https://mongoosejs.com/)
- [commander.js(命令行工具库)](https://github.com/tj/commander.js/)
- [Inquirer.js(命令行工具库)](https://github.com/SBoudrias/Inquirer.js/)

&nbsp;

## 启动 MongoDB

出于简单考虑，我的 MongoDB 会在公有云的云主机上以一个 docker 容器的形式启动(吃灰的主机终于可以派上用场了)。

首先拉取 MongoDB 的镜像

```code
docker pull mongo
```

启动镜像，将主机的 27017 端口映射至容器

```code
docker run -d -p 27017:27017 --name mongodb mongo
```

检查

```code
root@hecs-x-large-4-linux-20200521104533:~# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                      NAMES
a69390ba5ffe        mongo               "docker-entrypoint.s…"   3 hours ago         Up 3 hours          0.0.0.0:27017->27017/tcp   mongodb
```

创建数据库 todoapp

```code
root@hecs-x-large-4-linux-20200521104533:~# docker exec -it mongodb mongo
> use todoapp
switched to db todoapp
> show collections
```

&nbsp;

## 连接 MongoDB

启动成功后，通过 Node.js 连接 MongoDB，便可以进行增删改查操作，这次我使用的库是 Mongoose，Mongoose 使用 `Schema` 结构定义数据模型，封装完善，使用起来非常简单。

要使用 Mongoose，只需要在项目中安装依赖即可：

```code
$ npm i mongoose --save
```

依赖安装成功后，创建 `db.js`，用于初始化数据库连接以及封装对数据库的增删改查方法

```javascript
 /* db.js */
const mongoose = require('mongoose') //引用 mogoose

//通过 connect 方法连接到数据库，第一个参数是数据库的 url，最后一个 todoapp 是我的数据库名
mongoose.connect('mongodb://127.0.0.1:27017/todoapp', {
    //这两个 option 是必填的，不然会报错
    useNewUrlParser: true,
    useUnifiedTopology: true
})
//定义 Schema，跟数据库中的表字段对应
const taskSchema = mongoose.Schema({
    title: String,
    done: Boolean
})
//定义 Model，第一个参数是 Model 名称，第二个参数是 Schema，第三个参数是数据库中的表名称
const Task = mongoose.model('Task', taskSchema, 'tasks')
```

到这里，如果配置正确，端口没有阻塞，服务运行正常的话，实际上连接就已经成功了，我们定义了一个 `Schema`用来描述我们表（在 MongoDB 里叫做集合）中的字段，然后将 Schema 编译成 Model，接下来便可以通过 Model 类对数据库进行操作了。

&nbsp;

## 封装 db 对象

在 db.js 中将对数据库的操作封装在 db 对象中，并通过 module.exports 导出：

```javascript
 /* db.js */
const db = {
    //查找，为空即查找全部
    read(data = {}) {
        return new Promise((resolve, reject) => {
            Task.find(data).then(res => {
                resolve(res)
            }).catch((err) => {
                reject(err)
            })
        })
    },
    //保存
    write(data) {
        return new Promise((resolve, reject) => {
            //注意，保存操作需要对 Model 进行实例化，再调用save方法
            const t = new Task(data)
            t.save().then(() => {
                resolve()
            }).catch((err) => {
                reject(err)
            })
        })
    },
    //修改
    update(field,id, newData) {
        return new Promise((resolve, reject) => {
            Task.updateOne({_id: id}, {[field]: newData}).then(res=>{
                resolve(res)
            }).catch((err)=>{
                reject(err)
            })
        })
    },
    //根据 id 删除
    delete(id){
        return new Promise((resolve, reject) => {
            Task.deleteOne({_id: id}).then(res=>{
                resolve(res)
            }).catch((err)=>{
                reject(err)
            })
        })
    },
    //清除全部
    clear(){
        return new Promise((resolve, reject) => {
            Task.remove({}).then(res=>{
                resolve(res)
            }).catch((err)=>{
                reject(err)
            })
        })
    },
    close(){
        mongoose.connection.close()
    }
}
//将 db 对象导出
module.exports = db
```

注意，我们目前只是在 MongoDB 中创建了一个数据库「todoapp」，并没有在里面创建集合，不过其实不必担心，跟传统的关系型数据库不同，在新增数据时，MongoDB 会自动进行创建。

补充：[db.js 完整代码](https://gist.github.com/wumanho/4154b6307e3d2c626a2767f4c0f01fcc)

&nbsp;

## 实现 API

这一步是创建 `api.js`，引入 `db` 对象，并实现所有需要的操作

```javascript
/* api.js */
const db = require('./db')  //引入 db 对象
```

首先我们需要一个新增操作，接受一个任务标题，并写入到数据库中

```javascript
/* api.js */
//添加任务
module.exports.add = (title) => {
    db.write({title: title, done: false}).then(() => {
        console.log('添加成功')
        db.close()
    }).catch(err => {
        console.log('添加失败' + err)
        db.close()
    })
}
```

清空所有任务，直接调用 clear() 操作即可

```javascript
/* api.js */
//清空任务
module.exports.clear = () => {
    db.clear().then(() => {
        console.log('清除成功')
        db.close()
    }).catch((err) => {
        console.log('清除成功失败' + err)
        db.close()
    })
}
```

查看所有任务

```javascript
/* api.js */
//展示所有
module.exports.listAll = async () => {
    return await db.read()
}
```

更新标题以及更新状态，在 db.update() 方法中，我们通过 field 去判断更新的是标题还是状态

```javascript
/* api.js */
//更新标题
module.exports.updateTitle = (id, newTitle) => {
    db.update('title', id, newTitle).then(() => {
        console.log('更新标题成功')
        db.close()
    }).catch((err) => {
        console.log('更新标题失败' + err)
        db.close()
    })
}

//更新状态
module.exports.updateStatus = (id, newStatus) => {
    db.update('done', id, newStatus).then(() => {
        console.log('更新成功')
        db.close()
    }).catch((err) => {
        console.log('更新失败' + err)
        db.close()
    })
}
```

根据 ID 删除任务，这里多说一句，我们在定义 Schema 时并没有声明 id 字段，这个 id 字段也是 MongoDB 自动生成的。

```javascript
/* api.js */
//删除任务
module.exports.delete = (id) => {
    db.delete(id).then(() => {
        console.log('删除成功')
        db.close()
    }).catch(err => {
        console.log('删除失败' + err)
        db.close()
    })
}
```

再提供一个断开连接的方法，直接调用 db 的 close 方法即可

```javascript
/* api.js */
//断开连接
module.exports.close = () => {
    db.close()
}
```

补充：[api.js 完整代码](https://gist.github.com/wumanho/60498d5a5aaf188bda9ea7d7f70baed9)

&nbsp;

## 实现命令行功能

API 都写好了之后，就可以开始着手做命令行交互了。

现在我们需要用到文章开头提到的两个第三方库，`commander.js` 以及 `inquirer.js`。

其中 commander 是一个使 Node.js 实现命令行操作的库，它可以定义选项、参数、命令及子命令等丰富的命令行操作，而 inquirer 则用于帮助我们扩展命令行的交互功能。

安装依赖：

```code
npm i commander --save
npm i inquirer --save
```

创建 `c.js`，引入 api 以及两个库

```javascript
/* c.js */
const {Command} = require('commander');
const program = new Command();
const api = require('./api')
const inquirer = require('inquirer');
```

### 创建命令

接下来，通过实例化的 Command 对象 program 来创建一个顶层命令，顶层命令的意思是当用户没有输入任何选项或者命令时所需要实现的功能。

通过 program 对象的 .option() 方法定义了一个选项，如果用户在使用时添加了 -l 参数，则展示所有任务，否则就进入交互

```javascript
/* c.js */
program
    .option('-l, --list', '查看所有任务')
    .action(async (options, command) => {
    //在这里判断是否有 -l 参数
        if (options.list) {
            await listAll()
        } else {
            //后面实现
            await operation()
        }
    })
```

实现显示所有方法，如果发现表中没有数据，则对用户进行提示：

```javascript
/* c.js */
async function listAll() {
    const res = await api.listAll()
    if(res.length<1){
        console.log(`目前还没有数据，请使用 cli add<taskName> 命令添加`)
        api.close()
        return
    }
    res.forEach((task, index) => {
        console.log(`${res.done ? '[x]' : '[]'} ${index + 1} - ${task.title}`)
    })
    api.close()
}
```

通过 `program.command()` 方法实现 add 命令，add 命令接受一个必要的参数 taskName

```javascript
/* c.js */
program.command('add <taskName>')
    .description('添加一个任务，请勿使用空格')
    .action((taskName) => {
        api.add(taskName)
    })
```

定义 clear 命令，用于清空所有任务，不需要接受参数

```javascript
/* c.js */
program.command('clear')
    .description('清空所有任务')
    .action(() => {
        api.clear()
    })
```

最后别忘了使用 parse() 解析参数

```javascript
/* c.js */
program.parse(process.argv);
```

### 实现交互

交互功能在用户没有输入任何参数或命令的时候进入，定义 operation() 方法

```javascript
/* c.js */
async function operation() {
    let taskList = await api.listAll()
    if(taskList.length<1){
        console.log(`目前还没有数据，请使用 cli add<taskName> 命令添加`)
        api.close()
        return
    }
    inquirer.prompt(
        {
            type: 'list',
            name: 'index',
            message: '选择你的任务',
            choices: [...taskList.map((task, index) => {
                return {name: `${task.done ? '[x]' : '[_]'} ${index + 1} - ${task.title}`, value: index.toString()}
            }), {name: '+ 创建任务', value: '-2'}, {name: '退出', value: '-1'}]
        })
        .then((answer) => {
        	//这里实现选中后的交互功能
            askForAction(answer, taskList)
        });
}
```

inquirer 的 prompt() 方法用于弹出提示，用户可以对 choices 数组中的条目进行交互操作

用户可以选择对该任务进行一些列操作，也可以选择选择创建一个任务或者退出：

```javascript
async function askForAction(answer, taskList) {
    const index = parseInt(answer.index)
    if (index >= 0) {
        //选中了一个任务
        inquirer.prompt({
            type: 'list',
            name: 'action',
            message: '选择你的操作',
            choices: [
                {name: '标记为完成', value: 'done'},
                {name: '标记为未完成', value: 'notDone'},
                {name: '修改标题', value: 'update'},
                {name: '删除', value: 'delete'},
                {name: '退出', value: 'quit'},
            ]
        }).then(answer2 => {
            actions(answer2, index, taskList)
        })
    } else if (index === -2) {
        //创建任务
        inquirer.prompt({
            type: 'input',
            name: 'newTitle',
            message: '请输入任务名称',
        }).then(res => {
            api.add(res.newTitle)
        })
    }else{
        api.close()
    }
}
```

最后，当用户选中了具体的任务，则弹出「标记为完成」、「标记为未完成」、「修改标题」、「删除」、「退出」几个选项，我们需要实现每个选项的具体功能：

```javascript
async function actions(answer2, index, taskList) {
    switch (answer2.action) {
        case 'quit':
            api.close()
            break;
        case 'done':
            api.updateStatus(taskList[index]._id,true)
            break;
        case 'notDone':
            api.updateStatus(taskList[index]._id,false)
            break;
        case 'update':
            inquirer.prompt({
                type: 'input',
                name: 'newTitle',
                message: '新的标题',
                default: taskList[index].title
            }).then(res => {
                api.updateTitle(taskList[index]._id, res.newTitle)
            })
            break;
        case 'delete':
            api.delete(taskList[index]._id)
            break;
    }
}
```

至此，TODO 小工具的所有功能已经实现完成， 使用 node 命令直接执行 c.js 或者 c，就可以启动命令行了。

或者，可以直接将工具发布至 `npm`，供其他用户直接下载使用，具体方法就不在这边赘述了。

```code
node c
$ node c
? 选择你的任务 (Use arrow keys)
> [_] 1 - 练阅读
  [_] 2 - 练习听力
  + 创建任务
  退出

```

最后补上 [c.js 的完整代码](https://gist.github.com/wumanho/0a116247e31e905f8354cf163bf05df9)

&nbsp;

（完）

&nbsp;

## 参考

[mongoosejs官方文档](https://mongoosejs.com/)

[commanderjs 中文文档](https://github.com/tj/commander.js/blob/master/Readme_zh-CN.md)

[inquirerjs 仓库](https://github.com/SBoudrias/Inquirer.js/)

