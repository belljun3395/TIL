## Harilog_API

---



### Passport

---

어플을 만들때 가장 기본이 되는 것 중에 하나인 로그인 구현을 도와주는 라이브러리이다.

passport는 local 및 다양한 전략을 통해서 로그인을 하는 것을 도와준다.



우선 local 전략을 알아보자.

```js
const passport = require('passport');
const LocalStrategy = require('passport-local').Strategy;
const bcrypt = require('bcrypt');

var User = require('../../DB/sequelize/models/User');


module.exports = () => {
  passport.use(new LocalStrategy(
  {
    usernameField: 'email',
    passwordField: 'password',
  }, 
  async (email, password, done) => {
    try {
      const exUser = await User.findOne({ where: { email } });
      if (exUser) {
        const result = await bcrypt.compare(password, exUser.password);
        if (result) {
          done(null, exUser);
        } else {
          done(null, false, { message: '비밀번호가 일치하지 않습니다.' });
        };
      } else {
        done(null, false, { message: '가입되지 않은 회원입니다.' });
      };
    } catch (error) {
      console.error(error);
      done(error);
    }
  }
  ));
};
```



위는 local 전략을 구현한 코드이다.

```js
  {
    usernameField: 'email',
    passwordField: 'password',
  }, 
```

위와 같은 옵션을 통해 email과 password를 가지고 온다.

'email' 그리고 'password' 는 `usernameField` 그리고 `passwordField` 를 보면 알 수 있듯 로그인 폼의 name을 적어준 것이다.



이렇게 로그인 폼에서 email 그리고 password를 받으면 다음과 같이 활용한다.

```js
async (email, password, done) => { }
```

`{ }`  속의 코드는 DB와 유저 정보를 비교하여 확인하는 과정이다.

이때 비교과 성공한다면 다음과 같은 코드를 통해 이 정보를 `req.login(user, ...)` 으로 넘겨준다.



이렇게 넘겨 받은 정보를 통해 `passport.serializeUser` 와 `passport.deserializeUser` 와 같은 메서드를 수행한다.



> 추가적인 정보로 passport를 사용하지만 password를 사용하지 않는 경우가 있다.
>
> 이 경우에는 옵션을 다음과 같이 `usernameField` 와 `passwordField` 를 동일하게 설정하고 
>
> ```js
> {    usernameField: 'email',    passwordField: 'email',  }, 
> ```
>
> ```js
>   async (email, password, done) => { }
> ```
>
> 위와 같이 동일하게 이후 과정을 수행해주면 된다.



이렇게 설정한 local 전략은 `passport/index.js` 에 다음과 같이 등록해 준다.

```js
var passport = require('passport'),
    local = require('./localStrategy');

var User = require('../../DB/sequelize/models/User');

module.exports = () => {

    local();
    
    passport.serializeUser((user, done) => {
      done(null, user.id);
    });
    
    passport.deserializeUser((id, done) => {
      User.findOne({
        where: { id }
      })
      .then(user => done(null, user))
      .catch(err => done(err));
    });
};
```



이 `index.js` 역시 `app.js` 에 다음과 같이 연결하여 서버가 시작하기 전에 설정을 한다.

```js
passportConfig = require('./BackEnd/passport');

// config
passportConfig();

// express start
var app = express();
```



이렇게 `app.js` 에 등록을 하면 passport를 활용하여 다음과 같은 메서드를 만들어 활용할 수 있다.

1주차에는 `join`, `authenticate`, `isLoggedIn` 그리고 `isNotLoggedIn` 과 같은 메서드를 다음과 같이 구현하였다.

```js
var passport = require('passport');
const bcrypt = require('bcrypt');


var User = require('../../../DB/sequelize/models/User');


join = async (req, res, next) => {
    const { email, password, name, sex, cycle } = req.body;
    try {
      const exUser = await User.findOne({ where: { name } });
      if (exUser) {
        return res.redirect('/join?error=exist');
      }
      const hash = await bcrypt.hash(password, 12);
      await User.create({
        email,
        password : hash,
        name,
        sex,
        cycle,
      });
      return res.send("Join Success");
    } catch (error) {
      console.error(error);
      return next(error);
    }
  }

authenticate = (req, res, next) => {
    passport.authenticate('local', (authError, user) => {
      if (authError) {
        console.error(authError);
        return next(authError);
      };
      if (!user) {
        return res.send('NO EXISTING USER');
      };
      return req.login(user, (loginError) => {
        if (loginError) {
          console.error(loginError);
          return next(loginError);
        };
        return res.send("Login Success")
      });
    })(req, res, next);
};

isLoggedIn = (req, res, next) => {
  if (req.isAuthenticated()) {
      next();
  } else {
      res.status(403).send('로그인 필요');
  }
};

isNotLoggedIn = (req, res, next) => {
  if (!req.isAuthenticated()) {
      next();
  } else {
      const message = encodeURIComponent('로그인한 상태입니다.');
      res.redirect(`/?error=${message}`);
  }
};

deleteAPITest = async (req, res, next) => {
    await User.destroy({ where: { name:  "나나김"} });
    res.send("delete")
};


module.exports = { join , authenticate, isLoggedIn, isNotLoggedIn, deleteAPITest}

```



### Multer

---

multer은 multipart/form-data 형식으로 이미지와 같은 파일을 업로드 하는 것을 도와준다.



```js
var multer = require('multer'); // multer모듈 적용 (for 파일업로드)
var storage = multer.diskStorage({
  destination: function (req, file, cb) {
    cb(null, 'uploads/') // cb 콜백함수를 통해 전송된 파일 저장 디렉토리 설정
  }
  filename: function (req, file, cb) {
    cb(null, file.originalname) // cb 콜백함수를 통해 전송된 파일 이름 설정
  }
})
var upload = multer({ storage: storage })
```

위의 코드는 multer의 기본적인 실행 코드이다.

위의 코드를 multer를 사용할 라우터에 선언하고 사용하면 된다.



하지만 이는 다양한 라우터에서 사용하기에는 적절하지 않아 다음과 같이 수정하였다.

우선 `middlewares/multer` 폴더를 만들어 놓고 `index.js`  파일을 다음과 같이 만들었다.

```js
const multer  = require('multer'),
    path = require('path'),
    fs = require('fs');

class MulterClass {

    constructor() {
        try {
            fs.readdirSync('uploads');
            } catch (error) {
            console.error('uploads 폴더가 없어 uploads 폴더를 생성합니다.');
            fs.mkdirSync('uploads');
            }
            
        this.upload = multer({
            storage: multer.diskStorage({
                destination(req, file, cb) {
                    cb(null, 'uploads/');
                },
                filename(req, file, cb) {
                    const ext = path.extname(file.originalname);
                    cb(null, req.user.id + "_" + path.basename(file.originalname, ext) + Date.now() + ext);
                },
            }),
            limits: { fileSize: 5 * 1024 * 1024 },
        });
    }
        
    single(fieldName) {
        return this.upload.single(fieldName)
    }

    array(fieldName, count) {
        return this.upload.array(fieldName, count)
    }
    
    fields (arrayField) {
        return this.upload.fields(arrayField)
    }
     
}

    
module.exports = MulterClass;
```

기본적인 multer 사용코드와 차이는 multer 객체가 생성될 때 `upload` 폴더 유무를 확인하여 생성한다.



```js
var Multer = require("./index.js");

multerObj = new Multer();


// exports module
exports.single = (fieldName) => {
    return multerObj.single(fieldName);
}

exports.array = (fieldName, count) => {
    return multerObj.array(fieldName, count);
}

exports.fields = (array) => {
    return multerObj.fields(array);
}
```

그리고 위와 같이 `middlewares/multer` 폴더의 `multer.js` 파일을 만든다.

서버가 시작될 때 multer 객체가 생성되고 위와 같이 메서드들을 exports 한다.



그래서 아래와 같이 사용할 수 있다.

```js
router.post('/singleRecord', multer.single("formImage"))
```



### Record

---

```js
var User = require('../../../DB/sequelize/models/User')
var Designer = require('../../../DB/sequelize/models/Designer')
var Cut = require('../../../DB/sequelize/models/Cut')
var Perm = require('../../../DB/sequelize/models/Perm')
var Dyeing = require('../../../DB/sequelize/models/Dyeing')

var Post = {
    
    record  : async function(req, res) {
        var category = req.params.category
        var user = await User.findOne({where : {id : req.user.id}});
        var {record, image} = await Post.recordDesigner(req, category, user)
        var result = {record, image}
        result[`${category}`] = await Post.recordCategory(req, category, record)
        res.send(result)
    },
    
    recordDesigner :  async function(req, category, userInstance) {
        var {date, cost, time, designerName, etc, grade} = req.body;
        if(designer){
            var designer = await Designer.findOne({where : { designer : designerName }})
            var record = await userInstance.createRecord({ date, cost, time, category, etc, grade, DesignerId : designer.id })
            var image = await record.createImage({ img1 : req.file.filename})
            var result = {record, image}
            return result
        } else {
            var record = await userInstance.createRecord({ date, cost, time, category, etc, grade })
            var image = await record.createImage({ img1 : req.file.filename})
            var result = {record, image}
            return result
        }
    },
    
    recordCategory : async function(req, category, record) {
        switch (category) 
        {
            case "cut" : 
                var { cutName, cutLength } = req.body
                var cut = await Cut.create({ name :cutName , length : cutLength , RecordId : record.id})
                return cut;
            case "perm" : 
                var { permName, permTime, permHurt } = req.body 
                var perm = await Perm.create({name : permName, time : permTime, hurt : permHurt, RecordId : record.id})
                return perm
            case "dyeing" : 
                var { dyeingColor, dyeingDecolorization, dyeingTime, dyeingHurt } = req.body 
                var dyeing = await Dyeing.create({color : dyeingColor, dyeingDecolorization, time : dyeingTime, hurt : dyeingHurt, RecordId : record.id})
                return dyeing
        }
    },
    designer : async function(req, res) {
        var {designer, salon, fav} = req.body;
        var user = await User.findOne({wherer : {id : req.user.id}});
        var designerRecord = await user.createDesigner({designer, salon, fav})
        res.send(designerRecord)
    }

}

var Get = {
    record : async function(req, res) {
        var user = await User.findOne({wherer : {id : req.user.id}});
        var recordArray = await user.getRecords({raw : true});
        var recordObj = Object.assign({}, recordArray)
        var recordCount = await user.countRecords()
        var img = {};
        for (var i = 0; i < recordCount; i++) {
            img[i] = await Image.findOne({where : {RecordId : recordArray[i].id}, raw : true})
        }
        var result = {user, record : recordObj, img}
        res.send(result)
    }
}


var Update = {

}

module.exports = {Post, Get, Update};
```

위는 record와 관련된 미들웨어를 구현한 것이다.

위의 경우 sequelize의 인스턴스 메서드를 활용하여 구현하였다.



```js
var record = await userInstance.createRecord({ date, cost, time, category, etc, grade, DesignerId : designer.id })
```

대표적으로 위와 같은 코드가 있을 수 있다.

`userInstance` 는`var user = await User.findOne({where : {id : req.user.id}});` 를 통해 찾은 user 객체를 넘겨 받는다.

그 객체에는 sequelize가 연관관계가 있을 때 자동으로 제공하는 메서드가 있고 그것 중 하나가 `createRecord` 이다.

위와 같이 sequelize가 연관관계가 있을 때 자동으로 제공하는 메서드를 사용한 이유는 하드코딩을 할 때 발생할 수 있는 실수를 방지하기 위해서이다.

**하지만 추후에 sequelize를 사용하는 것이 아닌 sql query 문을 활용하여 다시 코드를 작성하는 것도 좋을 것 같다.**