## Steps to use cra with typescript and antd without eject

Based on https://github.com/comerc/cra-ts-antd and https://ant.design/docs/react/use-with-create-react-app

### CAUTION : this project only works with create-react-app-typescript 2.6.0 (react-scripts@1.0.10). It's apparently broken with create-react-app-typescript 2.7.0 (react-scripts@1.0.13). To investigate.

### Step 1

```
yarn global add create-react-app
```

### Step 2

```
create-react-app ant-demo-ts-cra --scripts-version=react-scripts-ts
```

Note : this commands installs react, react-dom, react-scripts and initialize the boilerplate.

### Step 3

```
cd ant-demo-ts-cra
yarn start
```

If you get the the following message :  

```
yarn start v0.27.5
$ react-scripts start
Something is already running on port 3000.
Done in 2.17s.
```

Create a .env file at the root of project, with the line `PORT=3001`
(see https://github.com/wmonk/create-react-app-typescript/issues/125 for more context)

Open http://localhost:3001 and check if you see the React logo.


### Step 4 : Use react-app-rewired to customize webpack without ejecting

```
yarn add react-app-rewired --dev
```

Modify the scripts field in package.json.

```diff
/* package.json */
"scripts": {
-   "start": "react-scripts start",
+   "start": "react-app-rewired start --scripts-version react-scripts-ts",
-   "build": "react-scripts build",
+   "build": "react-app-rewired build --scripts-version react-scripts-ts",
-   "test": "react-scripts test --env=jsdom",
+   "test": "react-app-rewired test --env=jsdom",
}
```

In order to override webpack config without ejecting, create a file config-overrides.js at the root and edit as following : 

```ts
module.exports = function override(config, env) {
	return config
}
```

*Note: we will add a real webpack configuration override later, we just let this file blank now to check if react-app-rewired is installed and if our react app is still ok.*

Run `yarn start` to check if everything is fine.

### Step 5 : install ant-design

This one is an easy one:  
`yarn add antd`

### Step 6 : use plugins and loaders

Now we need babel and some additional plugins to : 
- import antd components and according styles on demand
- have babel settings for react
- modify less variables
- transpile typescript/tsx to js, jsx

```
yarn add babel-core babel-plugin-import babel-preset-react-app awesome-typescript-loader react-app-rewire-less --dev
```

modify config-overrides.js like this : 

```diff
+const { injectBabelPlugin } = require('react-app-rewired');
+const rewireLess = require('react-app-rewire-less');
+
+module.exports = function override(config, env) {
+	const tsLoader = config.module.rules.find(conf => {
+		return conf.loader && conf.loader.includes('ts-loader')
+	})
+	tsLoader.loader = require.resolve('awesome-typescript-loader')
+	tsLoader.query = {
+		useBabel: true,
+	}
+
+	const tsLintLoader = config.module.rules.find(conf => {
+		return conf.loader && conf.loader.includes('tslint-loader')
+	})
+	tsLintLoader.options = tsLintLoader.options || {}
+	// FIXED Warning: The 'no-use-before-declare' rule requires type infomation.
+	tsLintLoader.options.typeCheck = true
+
+	const path = require('path')
+	// For import with absolute path
+	config.resolve.modules = [path.resolve('src')].concat(config.resolve.modules)
+
+	config = injectBabelPlugin(['import', { libraryName: 'antd', style: true }], config);  // +change importing css to less
+	config = rewireLess(config, env, {
+		//We use modifyVars option of less- loader here, you can see a green button rendered on the +page after reboot start server.
+		modifyVars: { "@primary-color": "#1DA57A" },
+	});
+
+	return config
+}
```

### Step 7 : edit tsconfig.json

```diff
+ "allowSyntheticDefaultImports": true
```

If you don't do this, you will get error:  
`JSX element type [WhicheverAntdComponentYouTryToUse] is not a constructor function for JSX elements.
  Property 'setState' is missing in type [xxx].`

### Step 8 : add .babelrc

Create this file at root of project:  

```diff
+ {
+ 	"presets": [
+ 		"react-app"
+ 	],
+ 	"plugins": [
+ 		[
+ 			"import",
+ 			{
+ 				"libraryName": "antd",
+ 				"style": false
+ 			}
+ 		]
+ 	]
+ }
```

### Step 9 : add file main.less : 

This file is needed to customize antd theme with less.  
Create the file src/resources/main.less with this line: 
```diff
+ import "~antd/dist/antd.less"; // import official less entry file
```


### Step 10 : test antd app : 

Modify App.tsx

```diff
import * as React from 'react';
+ import { Button } from 'antd';
import './App.css';

const logo = require('./logo.svg');

class App extends React.Component {
  render() {
    return (
      <div className="App">
-        <div className="App-header">
-          <img src={logo} className="App-logo" alt="logo" />
-          <h2>Welcome to React</h2>
-        </div>
-        <p className="App-intro">
-          To get started, edit <code>src/App.tsx</code> and save to reload.
-        </p>
+        <Button type="primary">Button</Button>
      </div>
    );
  }
}
```

You should see the button "Button".
The button should be gree, due to the use of modifyVar in config-overrides.js 
