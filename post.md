## Introduction

Hybrid apps are gaining popularity. The Gartner research firm [predicts](http://www.gartner.com/newsroom/id/2429815?) that by 2016, hybrid apps will constitute the majority of mobile apps. So it is good time to learn some modern technologies for building hybrid apps.

In this tutorial we are going to see how it is possible to build [Cordova](https://cordova.apache.org/) application with push notifications based on the [Ionic Framework](http://ionicframework.com/) and [ngCordova](https://github.com/driftyco/ng-cordova/) created by the Ionic team.

Ionic uses [AngularJS](https://angularjs.org/), so it is assumed you are familiar with it. 

## What are we building in this tutorial?

Unfortunately, there are a lot of bad things happening every day. So we are going to create an iOS application to show some good news. We will also create own server for sending push notifications using [apn](https://github.com/argon/node-apn), based on [Node.js](https://nodejs.org/) and [MongoDB](https://www.mongodb.org/) with a simple form to create a new post. 

The name of the application will be **PushNews**, but of course you can use your own.

## Prerequisites

1. Mac OS X
2. iOS Developer Program membership
3. Node.js/npm installed on your mac
4. MongoDB

For deploying we will use [Heroku](https://www.heroku.com/) and [Mongolab](https://mongolab.com/), so you need to have accounts there or you can use your own hosting instead.

## Install Ionic

Now we should install/update CLI for Cordova and Ionic, to have an opportunity to create new apps, add cordova plugins, etc.

Run this command in your Terminal:

```bash,linenums=true
$ npm install -g ionic cordova
```

## App Development

### Creating Ionic app

For creating and serving basic Ionic app with **tabs** starter template, run

```bash,linenums=true
$ ionic start pushNews tabs
$ cd pushNews
$ ionic serve
```

![create-app-id](https://dl.dropboxusercontent.com/u/17828362/ionic-push-notifications/07.png)



### Installing additional libs and setting up routes

```bash,linenums=true
$ bower install --save ngCordova angular-local-storage
```

Include `ng-cordova.min.js` and `angular-local-storage.min.js` in the `index.html`

```markup,linenums=true
<!-- index.html -->
...
<script src="lib/angular-local-storage/dist/angular-local-storage.min.js"></script>
<script src="lib/ngCordova/dist/ng-cordova.min.js"></script>
<script src="cordova.js"></script>
...
```

Inject as an Angular dependency:

```javascript,linenums=true
// www/js/app.js
...
angular.module('app', ['ngCordova', 'LocalStorageModule'])
...
```

We need to change routes now. There will be four of them:
- Login page
- All news list
- News view page
- User profile page

```javascript,linenums=true
// www/js/app.js
...
.config(function ($stateProvider, $urlRouterProvider) {
  $urlRouterProvider.otherwise('/login');
  $stateProvider
	.state('login', {
      url: '/login',
	  templateUrl: 'templates/login.html',
	  controller: 'LoginCtrl'
    })
	.state('tab', {
	  abstract: true,
	  templateUrl: "templates/tabs.html"
	})
	.state('tab.news', {
	  url: '/news',
	  views: {
		'tab-news': {
		  templateUrl: 'templates/tab-news.html',
		  controller: 'NewsCtrl'
		}
	  }
	})
	.state('tab.details', {
	  url: '/news/:id',
	  views: {
		'tab-news': {
		  templateUrl: 'templates/details.html',
		  controller: 'DetailsCtrl'
		}
	  }
	})
	.state('tab.profile', {
	  url: '/profile',
	  views: {
		'tab-profile': {
		  templateUrl: 'templates/tab-profile.html',
		  controller: 'ProfileCtrl'
		}
	  }
	});
});
```

### Building the Server

Let's create [Express](http://expressjs.com/) server. I will use Heroku to host it. Follow official [Getting Started](https://devcenter.heroku.com/articles/getting-started-with-nodejs#introduction) if you are not familar with Heroku. As an alternative you can use for development your localhost with [ngrok](https://ngrok.com/).


```javascript,linenums=true
// server.js
var express = require('express');
var app = express();
var bodyParser = require('body-parser');

require('./config')(app);
require('./models')(app);

var config = app.get('config');

app.use(function (req, res, next) {
  res.header('Access-Control-Allow-Credentials', true);
  res.header('Access-Control-Allow-Origin', req.headers.origin);
  res.header('Access-Control-Allow-Methods', 'GET,PUT,POST,DELETE');
  res.header(
    'Access-Control-Allow-Headers', 
    'X-Requested-With, X-HTTP-Method-Override, Content-Type, Accept'
  );
  if ('OPTIONS' === req.method) {
	res.status(200).end();
  } else {
	next();
  }
});

app.use(bodyParser.json());
app.use(bodyParser.urlencoded({
  extended: true
}));

var router = express.Router();
app.set('router', router);
app.use(router);

var http = require('http');
http.createServer(app)
  .listen(config.PORT, function () {
	console.log('app start on port ' + config.PORT);
  });

require('./routes')(app);
```

Models and routes we will describe later.

### Authorization and Push Notifications

There are multiple ways to manage auth process in Ionic apps. We will use the next one: user login with twitter, then we store returned data in localStorage and remove it on logout. If user exists in localStorage he is logged in, if not, we will redirect him to login page. **Don't forget to add CSRF or other protection for the real app.**

### Login with Twitter

To use twitter we must also add [sha1](https://github.com/Caligatio/jsSHA) library:

```bash,linenums=true
$ bower install --save jsSHA
```

```markup,linenums=true
<!-- www/index.html -->
...
<script src="lib/jsSHA/src/sha1.js"></script>
...
```

Install **inAppBrowser** plugin:

```bash,linenums=true
$ cordova plugin add https://git-wip-us.apache.org/repos/asf/cordova-plugin-inappbrowser.git
```

Also you have to create your own twitter app on [Twitter App Management](https://apps.twitter.com/) to get keys.

A little earlier we have set `login.html` as template for the login state, let's create it:

```markup,linenums=true
<!-- www/templates/login.html -->
<div class="row row-center">
  <div class="col">
    <h2 class="text-center">Good News</h2>

    <div class="padding-top">
      <button class="button button-block button-calm" ng-click="twitter()">
        <i class="icon ion-social-twitter"></i>&nbsp;Login with Twitter
      </button>
    </div>
  </div>
</div>
```

Login Controller:

It should be noticed, that any cordova plugin call should be wraped with the **deviceready** event. Or use **$ionicPlatform.ready()** available in Ionic.
```javascript,linenums=true
// www/js/controllers.js
...
.controller('LoginCtrl', function ($scope, $state, $cordovaOauth, 
                                   UserService, Config, $ionicPlatform,
                                   $ionicLoading, $cordovaPush) {
  if (UserService.current()) {
	$state.go('tab.news');
  }
  $scope.twitter = function () {
	$ionicPlatform.ready(function () {
	  $cordovaOauth.twitter(Config.twitterKey, Config.twitterSecret)
	    .then(function (result) {
		  $ionicLoading.show({
		    template: 'Loading...'
		  });
		  UserService.login(result).then(function (user) {
			if (user.deviceToken) {
			  $ionicLoading.hide();
			  $state.go('tab.news');
			  return;
			}

			$ionicPlatform.ready(function () {
			  $cordovaPush.register({
				badge: true,
				sound: true,
				alert: true
			  }).then(function (result) {
			    UserService.registerDevice({
			      user: user, 
			      token: result
			    }).then(function () {
				  $ionicLoading.hide();
				  $state.go('tab.news');
				}, function (err) {
				  console.log(err);
				});
			  }, function (err) {
				console.log('reg device error', err);
			  });
			});
		  });
		}, function (error) {
		  console.log('error', error);
		});
	});
  };
})
...
```

We have been using [$cordovaOauth](http://ngcordova.com/docs/plugins/oauth/).twitter() to allow user to login with twitter in our app. This plugin will do for us all rough work, but you should check [sources](https://github.com/driftyco/ng-cordova/blob/master/src/plugins/oauth.js#L481) to understand that there is no server on the iPhone and we can't set up callback to redirect twitter back to our app like we could do this on the web. So we should open a new window, parce a token and close the window manually.

To send push notification to our users, we must know **device token**. We will receive it using **$cordovaPush.register()** function. Cordova plugin should be installed:

```bash,linenums=true
$ cordova plugin add https://github.com/phonegap-build/PushPlugin.git
```


After getting user data, we send it to the server. Check [this](http://blog.pluralsight.com/angularjs-step-by-step-services) article by [Jim Cooper](http://blog.pluralsight.com/author/jimcooperps) to understand why we shouldn't do this in Controller. Let's create Service.

```javascript,linenums=true
// www/js/userService.js
(function () {
  function _UserService($q, config, $http, localStorageService, $state) {
	var user;
	function loginUser(post) {
	  var deferred = $q.defer();

	  $http.post(config.server + '/user/login', post)
	    .success(function (data) {
		  if (data.error || !data.user) {
		    deferred.reject(data.error);
		  }
		  localStorageService.set('user', data.user);
		  user = data.user;
		  
		  deferred.resolve(data.user);
	    })
	    .error(function () {
	      deferred.reject('error');
	    });
	    
	    return deferred.promise;
	}

	function logoutUser() {
	  localStorageService.remove('user');
	  user = null;
	  $state.go('login');
	}

	function currentUser() {
	  if (!user) {
		user = localStorageService.get('user');
	  }
	  return user;
	}

	function registerDevice(putData) {
	  var deferred = $q.defer();

	  $http.put(config.server + '/user/registerDevice', putData)
		.success(function (data) {
		  if (data.error || !data.user) {
			deferred.reject(data.error);
		  }

		  localStorageService.set('user', data.user);
		  user = data.user;

		  deferred.resolve(data.user);
		})
		.error(function () {
		  deferred.reject('error');
		});
		
	    return deferred.promise;
	}

	return {
	  login: loginUser,
	  logout: logoutUser,
	  current: currentUser,
	  registerDevice: registerDevice
	};
  }

  function _ConfigService() {
	return {
	  server: 'http://push-news.herokuapp.com',
	  twitterKey: 'your_twitter_key',
	  twitterSecret: 'your_twitter_secret'
	};
  }

  _UserService.$inject = [
    '$q', 'Config', '$http', 'localStorageService',
    '$state', '$cordovaPush', '$ionicPlatform'
  ];

  angular.module('app.services')
	.factory('UserService', _UserService)
	.service('Config', _ConfigService);
})();

 ```

Don't forget to include the new file in the `index.html`
```markup,linenums=true
<!-- index.html -->
...
<script src="js/userService.js"></script>
...
```

### Server side (user)

We should add user model and routes for **/user/login** and **/user/registerDevice** on our server

```javascript,linenums=true
// models/users.js 
var mongoose = require('mongoose');
var Schema = mongoose.Schema;

module.exports = function () {
  'use strict';

  var UsersSchema = new Schema({
	socialId: {type: String, index: true, unique: true},
	username: {type: String, unique: true},
	deviceToken: {type: String, unique: true},
	deviceRegistered: {type: Boolean, default: false},
	createdAt: {type: Date, default: Date.now}
  });

  mongoose.model('Users', UsersSchema, 'Users');
};
```

```javascript,linenums=true
// routes/user.js
var mongoose = require('mongoose');
var Users = mongoose.model('Users');

module.exports = function (app) {
  'use strict';

  var router = app.get('router');

  router.post('/user/login', function (req, res) {
	var socialId = req.body.user_id;
	var username = req.body.screen_name;

	Users.findOne({socialId: socialId}).lean().exec(function (err, user) {
	  if (err) {
		return res.json({error: err});
	  }

	  if (user) {
		return res.json({error: null, user: user});
	  }

	  var newUser = new Users({
		socialId: socialId,
		username: username
	  });
	  
	  newUser.save(function (err, user) {
		if (err) {
		  return res.json({error: err});
		}
		
		res.json({user: user, error: null});
	  });
	});
  });

  router.put('/user/registerDevice', function (req, res) {
	var user = req.body.user;
	var token = req.body.token;

	Users.findByIdAndUpdate(user._id, {
	  $set: {
		deviceToken: token,
		deviceRegistered: true
	  }
	}).lean().exec(function (err, user) {
	  if (err || !user) {
		return res.json({error: err});
	  }

	  res.json({error: null, user: user});
	});
  });
};
 ```

### News

In `app.js` we have set up already two routes for the news section: **tab.news** and **tab.details**.
On the first one we will see all news. Tapping on each of them will open view page with the news details.

### News list
```markup,linenums=true
<!-- www/templates/tab-news.html -->
<ion-view view-title="Good News">
  <ion-content>
    <ion-refresher
        pulling-text="Pull to refresh..."
        on-refresh="refresh()">
    </ion-refresher>
    <ul class="list">
      <li class="item" ng-repeat="n in news" ui-sref="tab.details({id: n._id})">
        {{n.text}}
      </li>
    </ul>
  </ion-content>
</ion-view>
```

To refresh news list we have used [pull to refresh](http://ionicframework.com/docs/api/directive/ionRefresher/), available in the Ionic and typical for native apps. 

```javascript,linenums=true
// www/js/controllers.js
...
.controller('NewsCtrl', function ($scope, NewsService, $ionicLoading) {
  $ionicLoading.show({
	template: 'Loading...'
  });
  NewsService.all().then(function (news) {
	$scope.news = news;
	$ionicLoading.hide();
  });

  $scope.refresh = function () {
	NewsService.all().then(function (news) {
	  $scope.news = news;
	  $scope.$broadcast('scroll.refreshComplete');
	});
  };
})
...
```

### Details page

```markup,linenums=true
<!-- www/templates/details.html -->
<ion-view view-title="Good News">
  <ion-content>
    <div class="card">
      <div class="item item-divider">
        {{news.createdAt | date : fullDate}}
      </div>
      <div class="item item-text-wrap">
        {{news.text}}
      </div>
      <div class="item item-divider">
        by {{news.username}}
      </div>
    </div>
  </ion-content>
</ion-view>
```

```javascript,linenums=true
// www/js/controllers.js
...
.controller('DetailsCtrl', function ($scope, $state, NewsService, 
                                     $ionicLoading) {
  $ionicLoading.show({
	template: 'Loading...'
  });
  var id = $state.params.id;
  NewsService.one(id).then(function (news) {
	$scope.news = news;
	$ionicLoading.hide();
  });
})
...
```

Let's create dedicated service for backend requests

```javascript,linenums=true
// www/js/newsService.js
(function () {
  function _NewsService($q, config, $http) {

	function getOne(id) {
	  var deferred = $q.defer();

	  $http.get(config.server + '/news/' + id)
		.success(function (data) {
		  if (data.error || !data.news) {
			deferred.reject(data.error);
		  }

		  deferred.resolve(data.news);
		})
		.error(function () {
		  deferred.reject('error');
		});
		
		return deferred.promise;
	  }

	  function getAll() {
	    var deferred = $q.defer();

		$http.get(config.server + '/news')
		  .success(function (data) {
			if (data.error || !data.news) {
			  deferred.reject(data.error);
			}

			deferred.resolve(data.news);
		  })
		  .error(function () {
			deferred.reject('error');
		  });
		  
		  return deferred.promise;
	  }

	  return {
		one: getOne,
		all: getAll
	  };
  }

  _NewsService.$inject = ['$q', 'Config', '$http'];

  angular.module('app.services')
	.factory('NewsService', _NewsService);
	
})();
```

### Server side (news)
We need a simple form for creating news on the server

```markup,linenums=true
<!-- adminpanel/add-news.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Add News</title>
</head>
<body>
  <form action="/news" method="post">
    <h2>Add good news</h2>
    <p>
      <textarea name="text" cols="50" rows="10" 
                placeholder="News body"></textarea>
    </p>
    <p>
      <input type="text" required name="username" 
             placeholder="Username"/>
    </p>
    <p>
      <button type="submit">Add news</button>
    </p>
  </form>
</body>
</html>
```

express.static middleware

```javascript,linenums=true
// server.js
 ...
app.use(express.static(__dirname + '/adminpanel'));
app.get('/add-news', function (req, res) {
  res.sendFile(__dirname + '/adminpanel/add-news.html');
});
 ...
 ```

![add-news](https://dl.dropboxusercontent.com/u/17828362/ionic-push-notifications/11.png)

Now we have to describe news model.

```javascript,linenums=true
// models/news.js
var mongoose = require('mongoose');
var Schema = mongoose.Schema;

module.exports = function () {
  'use strict';

  var NewsSchema = new Schema({
	username: String,
	text: String,
	createdAt: {type: Date, default: Date.now}
  });

  mongoose.model('News', NewsSchema, 'News');
};
```

On receiving a POST request we should create a new record in db and send push notifications with created news to all our users.

```javascript,linenums=true
// routes/news.js
var mongoose = require('mongoose');
var News = mongoose.model('News');
var Users = mongoose.model('Users');
var apn = require('apn');
var _ = require('lodash');

module.exports = function (app) {
  'use strict';

  var router = app.get('router');

  router.get('/news', function (req, res) {
	News.find().sort({createdAt: -1}).lean().exec(function (err, news) {
	  if (err) {
		return res.json({error: err});
	  }

	  res.json({error: null, news: news});
	});
  });

  router.get('/news/:id', function (req, res) {
	var id = req.params.id;
	News.findById(id).lean().exec(function (err, news) {
	  if (err) {
		return res.json({error: err});
	  }

	  res.json({error: null, news: news});
	});
  });

  router.post('/news', function (req, res) {
	var username = req.body.username;
	var text = req.body.text;

	var news = new News({
	  username: username,
	  text: text
	});

	news.save(function (err, news) {
	  process.nextTick(function () {
		sendPush(news);
	  });
	  res.redirect('/add-news');
	});
  });

  function sendPush(news) {
	var text = news.text.substr(0, 100);
	Users.find({deviceRegistered: true}).lean().exec(function (err, users) {
	  if (!err) {
		for (var i = 0; i < users.length; i++) {
		  var user = users[i];

		  var device = new apn.Device(user.deviceToken);
		  var note = new apn.Notification();
		  note.badge = 1;
		  note.contentAvailable = 1;
		  note.alert = {
			body : text
		  };
		  note.device = device;

		  var options = {
			gateway: 'gateway.sandbox.push.apple.com',
			errorCallback: function(error){
			  console.log('push error', error);
			},
			cert: 'PushNewsCert.pem',
			key:  'PushNewsKey.pem',
			passphrase: 'superpass',
			port: 2195,
			enhanced: true,
			cacheLength: 100
		  };
		  var apnsConnection = new apn.Connection(options);
		  console.log('push sent to ', user.username);
		  apnsConnection.sendNotification(note);
		}
	  }
	});
  }
};
```
We have used [apn](https://github.com/argon/node-apn) module to send push notifications. Also we used `PushNewsCert.pem` and `PushNewsKey.pem` files, we will create them in the next step.

Push notification will be sent, but how app will know about it? We must add an event listener to the `app.js`

```javascript,linenums=true
// www/js/app.js
...
.run(function ($ionicPlatform, $rootScope) {
  $ionicPlatform.ready(function () {
	$rootScope.$on(
	  '$cordovaPush:notificationReceived', 
	  function (event, notification) {
		if (notification.alert) {
		  navigator.notification.alert(notification.alert);
		}
	  });
  });
})
...
```

## Provisioning Profile and Certificates

For using push notifications in our app, we have to prepare the App ID, SSL Certificate and create Provisioning Profile in iOS Dev Center.

### Create Certificate

Open Keychain Access on your mac and choose **Request a Certificate from a Certificate Authority...**

![request-certificate](https://dl.dropboxusercontent.com/u/17828362/ionic-push-notifications/01.png)

Enter you email, common name, check Saved to disk. Save file as `PushNews.certSigningRequest`

![enter-data](https://dl.dropboxusercontent.com/u/17828362/ionic-push-notifications/02.png)

Now we can find new private key in keychain, let's export it as `PushNewsKey.p12`. Enter secure passphrase when it will prompt.

![export-key](https://dl.dropboxusercontent.com/u/17828362/ionic-push-notifications/03.png)

Log in to the [iOS Dev Center](https://developer.apple.com/ios) and select the **Certificates, Identifiers and Profiles** from the right panel.

Select **Identifiers** in the **iOS Apps** section.

Go to **App IDs** in the sidebar and click the **"+"** button.

![create-app-id](https://dl.dropboxusercontent.com/u/17828362/ionic-push-notifications/04.png)

Enter App ID registration data:

* App ID Description / Name - **PushNews**
* Explicit App ID / Bundle ID - **com.your_domain.PushNews** (this ID and cordova app name should be the same)
* App Services / Enable Services - **Push Notifications**

Press **Continue** and **Submit**.

Now we have to set up our app.
Select **PushNews** in Apps IDs list and press **Edit** in the bottom.

Find **Push Notifications** section and press **Create Certificate...**
in the **Development SSL Certificate** section.

![create-app-id](https://dl.dropboxusercontent.com/u/17828362/ionic-push-notifications/05.png)

Press **Continue** and Choose `PushNews.certSigningRequest` created early, press **Generate**

Download generated certificate and save it as `aps_development.cer`

### Making PEM files

Go to the directory where you've saved files and run these commands in **Terminal** for converting: 

**.cer** file into a **.pem**

```bash,linenums=true
$ openssl x509 -in aps_development.cer -inform der -out PushNewsCert.pem
```
**.p12** into a **.pem**
```bash,linenums=true
$ openssl pkcs12 -nocerts -in PushNewsKey.p12 -out PushNewsKey.pem

```
You will be asked to enter your passphrase for **.p12** file and new pass for **.pem** file. Copy both **.pem** files to the app's root directory.

### Making the Provisioning Profile

Let's return to iOS Dev Center. Click the **Provisioning Profiles**/**Development** in the sidebar and click the **"+"** button.

* Choose **iOS App Development**
* Select **PushNews** App in App ID list
* Select your certificate
* Select devices you want to include in this provisioning profile
* Enter Profile Name (I will use "PushNews Development")

Press **Generate** and download profile when it will be created, we will use it later.

![provisioning](https://dl.dropboxusercontent.com/u/17828362/ionic-push-notifications/06.png)

Let's change cordova id in `config.xml` (by default it will be something like com.ionicframework.PushNews456803) to **Explicit App ID / Bundle ID** value you have entered above (com.your_domain.PushNews).

```markup
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<widget id="com.telnov.PushNews" version="0.0.1" xmlns="http://www.w3.org/ns/widgets"
        xmlns:cdv="http://cordova.apache.org/ns/1.0">
    <name>pushNews</name>
    ...
```

## Test on device

OK, now you can check result. To launch app on iOS device, we have to add ios platform first:

```bash,linenums=true
$ ionic platform add ios
$ ionic build ios
```
- Open `platforms/ios/pushNews.xcodeproj` in the Xcode
- Connect your iOS device via usb
- Switch to the **Build Settings** tab
- Select your **Provisioning Profile**
- Select your device
- Click run

![xcode](https://dl.dropboxusercontent.com/u/17828362/ionic-push-notifications/13.png)

You will see login page. After tapping the **Login with Twitter** button, you will see twitter auth page.

![login-page](https://dl.dropboxusercontent.com/u/17828362/ionic-push-notifications/08.jpg)
![login-page](https://dl.dropboxusercontent.com/u/17828362/ionic-push-notifications/09.jpg)

At the first time, when you are trying to launch any application that use push notifications, you should give an access, so after you entering twitter credentials, you will see popup

![push-access](https://dl.dropboxusercontent.com/u/17828362/ionic-push-notifications/10.png)


Try adding news and enjoy the results.

![push](https://dl.dropboxusercontent.com/u/17828362/ionic-push-notifications/12.png)


## Conclusion
We have created the project, which demonstrates how easy you can build iOS applications using web technologies only. 

Full project you can find on the [Github](https://github.com/otelnov/pushNews).

Thanks for reading!

