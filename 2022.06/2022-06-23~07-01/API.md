## Harilog_API

---



### 리펙토링

---

1주차에는 주로 기능을 구현하는데 초점을 맞추고 코딩을 진행하였던 것 같다.

그래서 가장 중요하다고 생각하는 쓰기(C) 그리고 읽기(R)에 관한 코드를 우선 작성하였다.

코드의 경우는 문제가 없었는데 **async/await 를 많이 사용하는 것** 그리고 **중복되는 코드가 많다는 문제**가 있었다.



중복되는 코드의 경우 대부분이 매개변수로 받는 것이 다른 코드들이어서 매개변수의 공통분모를 잘 정하는 것만 신경쓰면 되어 크게 문제되지는 않았다.

```js
//classifyCategory.js 

// function latest
let img = await imgObj(recordArray, recordCount)

// function category
let img = await imgObj(recordObj)

// funtion categoryResult
let img = await imgObj(recordObj, 1)

const imgObj = async function(record, recordCount) {

    let img = await Promise.all(record.rows.map(
        rows => Image.findOne({where : {RecordId : rows.id}, raw : true})
    ))
    if(recordCount == 1){
        let img = await Image.findOne({where : {RecordId : record.rows[0].id}, raw : true})
    }
    return img = { count : record.count, rows : img}
    
}
```

위에서 보이는 것처럼 매개변수로 들어가는 것들이 조금씩 다르지만 그 틀을 맞추어 놓았기 때문에 `imgObj` 를 사용하는데 문제가 없었다.



async/await 를 많이 사용하는 문제는 async/await 가 서로 연관을 가지는 경우가 아니라면 `Promise.all()`을 사용하여 비동기적으로 문제를 해결하려 하였다.

```js
//classifyCategory.js

// function classify
let categoryRecord = await Promise.all(recordObj.rows.map(
    rows => Cut.findOne({where : {RecordId : rows.id}})
))
```

위와 같은 코드처럼`Promise.all()` 을 사용하면 메인 기록과 연관이 있는 Cut 기록 찾는 요청을 한번에 할 수 있게 된다.

이는 각각의 메인 기록이 서로 연관이 없기에 가능한 것이다.

 아래의 경우를 보자.

```js
// record.js

record  : async function(req, res) {
    let category = req.params.category
    let user = await User.findOne({where : {id : req.user.id}});
    let record = await Post.recordWithDesigner(req, category, user)
    let urls = await imageFunction.urls(req.files)
    let query = await imageFunction.query(urls)
    let image = await record.createImage(query)
    let result = {record, image}
    result[`${category}`] = await Post.recordCategory(req, category, record)
    res.send(result)
},
```

위의 경우는 user, record, urls, query, image가 각각 연관이 되어있기 때문에 `Promise.all()` 을 통해 한번에 요청하지 못한다.

`Promise.all()` 은 한번에 위의 요청을 보내는데 이는 순서를 보장하지 않기에 연관 관계가 있는 경우 사용하지 못한다.



### 기록이 없을 경우

---

기록이 없는 경우는 sequelize의 메서드를 활용하여 해결하였다.

sequelize가 제공하는 메서드에는 `findAllandCount` 라는 메서드가 있다.

이는 원하는 정보를 모두 찾아주는 것은 물론 그 개수까지 알려준다.

그렇기에 만약 기록이 없다면 0을 반환할 것이고 나는 이를 아래처럼 이용하기로 했다.

```js
//classifyCategory.js

if(dessignerList.count != 0) {
    let recordWithDesigner = await Promise.all(dessignerList.rows.map( 
        rows => Record.findAll({where : {DesignerId : rows.id}})
    ));
    
    let img = await Promise.all(dessignerList.rows.map(
        rows=> Image.findOne({where : {RecordId : rows.id}, raw : true})
    ));
    let result = {user, record : recordWithDesigner, img}
    return res.send(result)
}
```



### 리사이징

----

리사이징의 경우 npm의 sharp를 사용해서 리사이징을 하기로 했다.

리사이징을 하려한 이유는 클라우드에 이미지를 올리는데 너무 많은 시간이 소요되는 문제가 발생했기 때문이다.

나는 다음과 같이 sharp 코드를 작성하였다.

```js
// sharp.js

const sharp = require('sharp')
const fs = require('fs');

const sharping = async function(req, res, next) {

            return await Promise.all(
                req.files.map(file => 
                sharp(file.path)
                .resize({ width: 600 })
                .withMetadata()
                .toBuffer({resolveWithObject : true})
                .then(({data}) => {
                    fs.writeFile(file.path, data, (err) => {
                        if (err)
                            throw err;
                    })
                })
            )).then(()=> {
                return next()
            })

}


module.exports = {sharping}
```



사실 위의 코드를 작성하기 전에 문제가 있었다.

문제는 비동기적 처리의 순서 보장이었다.

```js
const sharp = require('sharp')
const fs = require('fs')

const sharping = async function(req, res, next) {

            await Promise.all(req.files.map(file => 
                sharp(file.path)
                .resize({ width: 600 })
                .withMetadata()
                .toBuffer((err, buffer) => {
                    if (err)
                        throw err;
    
                    fs.writeFile(file.path, buffer, (err) => {
                        if (err)
                            throw err;
                    });
                })
            ))


}


module.exports = {sharping}
```

처음 생각에는 `Promise.all()` 그리고 `map` 을 사용하고 `fs.writeFile` 이 콜백함수에 있기 때문에 문제가 없을 것이라 생각했다.

하지만 실제로는 그렇지 않았다.

`Promise.all()` 은 `buffer` 을 반환하는 것 까지만 순서를 보장하였다.

그렇기에 `fs.writeFile` 의 순서가 이후 코드들에 의해 보장되지 않았다.

그래서 `fs.writeFile` 의 순서를 보장 해줄 필요가 있었고 처음의 코드처럼 `.then` 을 통해 순서를 보장해주니 문제가 해결되었다.



이에 대한 추가적인 자료는 발표자료를 참고하면 될 것 같다.



### 클라우디너리

---

클라우디너리는 아마존 s3와 같은 클라우드에 이미지를 올리고 url을 반환해주는 클라우드이다.



클라우디너리를 사용하기 위한 코드는 다음과 같다.

```js
// cloudinary/config.js

const cloudinary = require('cloudinary').v2

exports.config = () => { cloudinary.config({
    cloud_name : process.env.CLOUD_NAME,
    api_key : process.env.API_KEY,
    api_secret : process.env.API_SECRET,
}) 
}
```

```js
// cloudinary/upload.js

const cloudinary = require('cloudinary').v2

const upload = async function (path){
    
    try{
        var result = await cloudinary.uploader.upload(path, {upload_preset: "hairlog"})
        return result.secure_url
    }catch(e){ 
        console.log(e)
    }

}


module.exports = { upload }
```

```js
// image.js

// function urls
let url = await Promise.all(images.map(result => cloudinary.upload(result[1])))
```

config.js를 통해 클라우디너리와 연결하는 코드를 작성하였고 upload.js를 통해 특정파일을 클라우디너리에 연결하는 코드를 작성하였다.

`cloudinary/upload.js` 의 경우 `image.js` 에서 `Promise.all()` 을 통해 비동기적으로 실행할 것이다.

