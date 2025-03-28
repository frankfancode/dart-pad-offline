# make dartpad run complete local
[中文](README-ZH.md)


TLDR: If you want to run dartpad-local, you can directly go to the section 
[Using dartpad-local](#Using-dartpad-local)

The official DartPad source code does not support complete offline usage. Therefore, we need to modify some source code, mainly by changing the host address and placing external resources locally.
Let's begin.


First, clone the DartPad code:

```bash
git clone https://github.com/dart-lang/dart-pad.git
```

Look at the directory structure:

```bash
tree -L 2
.
├── AUTHORS
├── CONTRIBUTING.md
├── LICENSE
├── README.md
├── doc
│   └── Sunflower.png
└── pkgs
    ├── dart_pad
    ├── dart_services
    ├── samples
    └── sketch_pad

```



To localize DartPad, we mainly need to modify dart-pad/pkgs/dart_pad and dart-pad/pkgs/dart_service.

dart_pad is the frontend to display and preview the code, 

dart_service is the backend to receive and run the code uploaded from the frontend and return the execution results.

You can refer to the official development documentation [CONTRIBUTING.md](https://github.com/dart-lang/dart-pad/blob/main/CONTRIBUTING.md) to see how to run it.

Go to the dart_pad directory and start the unmodified DartPad page:

```
$ dart pub get
$ dart ./tool/grind.dart serve
```
> 
> **Important Note: Each execution of dart `./tool/grind.dart serve` will trigger an internal build command that re-downloads and rebuilds the code. This is not .**
> You can add a run method in `grind.dart` and then just execute `dart ./tool/grind.dart run`.
> 


```dart
/// dart-pad/pkgs/dart_services/tool/grind.dart
@Task()
Future<void> run() async {
  await _run(Platform.executable, arguments: [
    path.join('bin', 'server.dart'),
    '--channel',
    _channel,
    '--port',
    '8082',
  ]);
}
```


After a few seconds, we will see the following output in the terminal, indicating that the DartPad frontend is now running locally:

``` 
...
...
  build/scripts/embed_html.dart.js compiled to 540k
  build/scripts/embed_inline.dart.js compiled to 538k
  Removed 0 Dart files

serve

finished in 42.2 seconds
Serving at http://localhost:8000
```

Open `http://localhost:8000` and you will see the interface below. The default page is Dart, you can try switching to the Flutter page by modifying sample. The processing and preview of Flutter programs is more complex than Dart programs.


| dart                                |  flutter                             |
| ----------------------------------- | ----------------------------------- |
| ![](attachment/63592f2ee4db89defd58171e3de5a278.png) | ![](attachment/7fe029d5cf9bce009a1c6b133e23d286.png) |


 
Although you can see DartPad running locally by opening `http://localhost:8000`, many resources are still fetched over the network. You can see these in console:
 ![](attachment/1fcbc091d7521012cae6f85c5118006d.png)

You can see that many resources are fetched from the internet. The internet resources required for just Dart functionality and Flutter functionality are different. Let's first make Dart completely offline.


## Localize Dart

### Remove google analytics

First, analytics are useless for a localized version, so we can remove them.
Search and delete the following code:

```html
<script src="scripts/ga.js" defer></script>
```

I deleted the references to ga in these files:

```
pkgs/dart_pad/test/embed/embed_test.html      |   1 -
pkgs/dart_pad/web/embed-dart.html             |   1 -
pkgs/dart_pad/web/embed-flutter.html          |   1 -
pkgs/dart_pad/web/embed-flutter_showcase.html |   1 -
pkgs/dart_pad/web/embed-html.html             |   1 -
pkgs/dart_pad/web/embed-inline.html           |   1 -
pkgs/dart_pad/web/index.html 
```


Restart the Dartpad service and look at the dev console again. There are no more requests to google analytics.
(If there is no effect, delete the build directory and force refresh the browser or switch new browsers.)

![](attachment/0db923e18710d63611554a186ee43d8c.png)

### Use a local DartService server 

From the dev console we can see that the backend server uses `api.dartpad.dev`.
We change the command so that dartpad can point to the local server:

```
In the dart-pad/pkgs/dart_pad directory, run this command
$ dart ./tool/grind.dart serve-local-backend
```
This command can be found in `dart-pad/pkgs/dart_pad/tool/grind.dart`.
In the Dev Console we can see that the DartPad server is now changed to `127.0.0.1` and `localhost`. And the port for the `compileDDC` request is `8082`. So our DartService also needs to provide service on port `8082`.


![](attachment/288113d8a3b15b94e4594da37a5fea33.png)
![](attachment/7fe773eeea8c921918d9285387951dad.png)
Next we start `dart_services`

```bash
cd dart-pad/pkgs/dart_services
dart pub get
# this step requires good network and is time consuming
FLUTTER_CHANNEL="stable" dart tool/grind.dart serve
```
After a long wait, you can see the following output indicating that the dart_service has started and is listening on port `8082`
```
.....
serve
  Running /Users/frank/fvm/versions/stable/bin/cache/dart-sdk/bin/dart bin/server.dart --channel stable --port 8082
  warning: no redis server specified.
  [info] Initializing dart-services:
  port: 8082
  sdkPath: /Users/frank/workspace/opensourcecode/dartpad/newdartpad/dart-pad/pkgs/dart_services/flutter-sdks/stable/bin/cache/dart-sdk
  redisServerUri: null
  Cloud Run Environment variables:
  [severe] PK_GITHUB_OAUTH_CLIENT_ID environmental variable not set! This is REQUIRED.
  [severe] PK_GITHUB_OAUTH_CLIENT_SECRET environmental variable not set! This is REQUIRED.
  [severe] GitHub OAuth Handler DISABLED - Ensure all required environmental variables are set and re-run.
  [info] Starting analysis server (sdk: flutter-sdks/stable/bin/cache/dart-sdk, args: --client-id=DartPad)
  [info] Analysis server initialized.
  [info] Listening on port 8082
```

Open `http://localhost:8000/` and you can see it is now processed by the local server:
![](attachment/e4814c6b9002cd5f68bd6beb5fd6685f.png)

Disconnect the network and test if it really works offline.
It functions normally but the page layout is messed up. Looking at the console shows that the font packages were not downloaded.
![](attachment/a06822a94de3c9f3fdf9decc48042ed2.png)
We need to change the fonts to load locally.
![](attachment/84b4295d0bf3de14f6e36242d7b4dbad.png)

Search and delete code that references remote fonts.
`https://fonts.googleapis.com/css2?`  
`https://fonts.googleapis.com/icon`
`https://fonts.googleapis.com/css`

Then download the fonts locally. You can find out which files to download by opening the URL that references the fonts, for example this one in the image above:
`https://fonts.googleapis.com/icon?family=Material+Icons`
```
/* fallback */
@font-face {
  font-family: 'Material Icons';
  font-style: normal;
  font-weight: 400;
  src: url(https://fonts.gstatic.com/s/materialicons/v140/flUhRq6tzZclQEJ-Vdg-IuiaDsNcIhQ8tQ.woff2) format('woff2');
}

.material-icons {
  font-family: 'Material Icons';
  font-weight: normal;
  font-style: normal;
  font-size: 24px;
  line-height: 1;
  letter-spacing: normal;
  text-transform: none;
  display: inline-block;
  white-space: nowrap;
  word-wrap: normal;
  direction: ltr;
  -webkit-font-feature-settings: 'liga';
  -webkit-font-smoothing: antialiased;
}
....
....
....
```

Download the woff2 file to a local directory, I put it in `dart-pad/pkgs/dart_pad/web/font/`. Then put the code in `dart-pad/pkgs/dart_pad/web/styles/styles.scss`:


```scss
// Added code in styles.scss:
@font-face {
font-family: 'Material Icons';
font-style: normal;
font-weight: 400;
src: url(../font/flUhRq6tzZclQEJ-Vdg-IuiaDsNcIhQ8tQ.woff2) format('woff2');
}

.material-icons {
font-family: 'Material Icons';
font-weight: normal;
font-style: normal;
font-size: 24px; /* Preferred icon size */
display: inline-block;
line-height: 1;
text-transform: none;
letter-spacing: normal;
word-wrap: normal;
white-space: nowrap;
direction: ltr;
/* Support for all WebKit browsers. */
-webkit-font-smoothing: antialiased;
/* Support for Safari and Chrome. */
text-rendering: optimizeLegibility;
/* Support for Firefox. */
-moz-osx-font-smoothing: grayscale;
/* Support for IE. */
font-feature-settings: 'liga';
}
```
Restart DartPad, now even offline it can work normally with localized DartPad website.
![](attachment/0a8b43e9f2522fbad385228906600d02.png)

If your goal is just to make Dart work offline and you don't need Flutter, then you have achieved your goal! 🎉


## Localize Flutter
More modifications are needed to make Flutter work locally, because more resources are needed to render the UI. From the requests in the console you can see some other resources are requested:

```bash
# Note the version number in your URL may be different than here  
https://storage.googleapis.com/nnbd_artifacts/3.1.2/dart_sdk.js
https://storage.googleapis.com/nnbd_artifacts/3.1.2/flutter_web.js

# Note the hash value in your URL may be different than here
https://www.gstatic.com/flutter-canvaskit/9064459a8b0dcd32877107f6002cc429a71659d1/chromium/canvaskit.js
https://www.gstatic.com/flutter-canvaskit/9064459a8b0dcd32877107f6002cc429a71659d1/chromium/canvaskit.wasm
https://fonts.gstatic.com/s/roboto/v20/KFOmCnqEu92Fr1Me5WZLCzYlKw.ttf

```
![](attachment/aef5be9b3b935e806cc5590d3cba441f.png)
We need to modify the code to request these resources locally instead.
First put the canvas related content locally:
Modify `dartpad/newdartpad/dart-pad/pkgs/dart_pad/lib/sharing/editor_ui.dart`:

``` dart
static String _createCanvasKitBaseUrl(String engineSha) {
	// const baseUrl = 'https://www.gstatic.com/flutter-canvaskit/';
	const baseUrl = '../flutter-canvaskit/';
	return path.join(baseUrl, '$engineSha/');
}

```

Put the `canvaskit.js` and `canvaskit.wasm` seen in the console into `dart-pad/pkgs/dart_pad/web/flutter-canvaskit`. The directory structure is:

```
tree
flutter-canvaskit
└──────────────── 9064459a8b0dcd32877107f6002cc429a71659d1
					└── chromium
						├── canvaskit.js
						└── canvaskit.wasm

3 directories, 2 files
```

Restart DartPad and see that it now fetches the `canvaskit` files locally. And the preview works normally.

![](attachment/b0b3cbd6fb392795dacabc665f979993.png)

Also put `dart_sdk.js` and `flutter_web.js` locally:

```
dart-pad/pkgs/dart_services
├── static_resource
   └── nnbd_artifacts
       └── 3.1.2
           ├── dart_sdk.js
           └── flutter_web.js
```

Since there are static resources, there needs to be a static resource server. These two resources were originally on `storage.googleapis.com`, we can put them under a dedicated static resource server. For simplicity, we manage these two resource files in dart_services and also start the static service in dart_services.
In `dart_service/pubspec.yaml` add dependency `shelf_static: ^1.1.2`
In `dart-pad/pkgs/dart_services/lib/server.dart` add support for static resources:


```dart
EndpointsServer._(String? redisServerUri, Sdk sdk) {
    ...

	/// handle static resource
    final staticResoucePath =
        path.join(Directory.current.path, 'static_resource');
    final staticHandler =
        createStaticHandler(staticResoucePath, listDirectories: true);

    pipeline = const Pipeline()
        .addMiddleware(logRequestsToLogger(_logger))
        .addMiddleware(createCustomCorsHeadersMiddleware());

	/// add static handler
    final cascade = Cascade();
    handler = pipeline.addHandler(
        cascade.add(staticHandler).add(commonServerApi.router.call).handler);
  }
```

Restart and you can see the Flutter program rendering normally. But there is still one resource being requested from `font.gstatic.com`. We change that to request from our local server in `dart_sdk.js`.

![](attachment/701b9358c7f5267e767e48c0b42cab51.png)

Open the local `dart_sdk.js` file and find:
```js
/*_engine._robotoUrl*/get _robotoUrl() {
      return "https://fonts.gstatic.com/s/roboto/v20/KFOmCnqEu92Fr1Me5WZLCzYlKw.ttf";
    },
```

Change to:
```js
/*_engine._robotoUrl*/get _robotoUrl() {
      return "../font/KFOmCnqEu92Fr1Me5WZLCzYlKw.ttf";
    },
```
put `KFOmCnqEu92Fr1Me5WZLCzYlKw.ttf` in `dart-pad/pkgs/dart_pad/web/font` 。


Restart `dart_pad` and `dart_service`, disconnect internet, then open `http://localhost:8000/`. Now Flutter can preview normally when offline.

## Github sample
If you want to also put Github samples locally, or other code locally, you can refer to this section.

```dart
/// Open pkgs/dart_pad/lib/sharing/gists.dart
/// Modify _gistApiUrl as follows: 
static const String _gistApiUrl = './gists';
```

put gist code here ``
```
dart-pad/pkgs/dart_pad/web/
├── gists
│   ├── 493c8b3ef8931cbac3fbbe5c04b9c4cf
│   ├── 4a546fc44db8aca351bfe791e251acc2
│   ├── 4a68e553746602d851ab3da6aeafc3dd
│   ├── 5c0e154dd50af4a9ac856908061291bc
│   ├── 85e77d36533b16647bf9b6eb8c03296d
│   ├── a133148221a8cbacbcef8bc77a6c82ec
│   ├── a1d5666d6b54a45eb170b897895cf757
│   ├── c0f7c578204d61e08ec0fbc4d63456cd
│   ├── d3bd83918d21b6d5f778bdc69c3d36d6
│   ├── d57c6c898dabb8c6fb41018588b8cf73
│   ├── e75b493dae1287757c5e1d77a0dc73f1
│   ├── ecabed4a17a3aad8bee7c6327e472fc8
│   ├── ef06ab3ce0b822e6cc5db0575248e6e2
│   └── fdd369962f4ff6700a83c8a540fd6c4c
```


Now, Dartpad can be run completely locally without the need for internet connection.
![](attachment/3634b2f7b5b0396b527e70b20244c5fc.png)



## Using dartpad-local

There are many modifications made. I have uploaded the modified version to https://github.com/frankfancode/dartpad-local
Except for the hash names of resources in flutter-canvaskit that need to be modified according to the local situation. You can use it by executing the following command.

```bash
# start dart_pad
cd dart-pad/pkgs/dart_pad
dart ./tool/grind.dart serve-local-backend

# start dart_services
cd dart-pad/pkgs/dart_services
# first time run it
FLUTTER_CHANNEL="stable" DART_SERVVICE_HOST_PATH="http://127.0.0.1:8082" dart tool/grind.dart serve

# after first run it
FLUTTER_CHANNEL="stable" DART_SERVVICE_HOST_PATH="http://127.0.0.1:8082" dart tool/grind.dart run

```


## Lastly
If there are any issues, please contact me.




## Ref
[How to create a custom DartPad?] (https://medium.com/flutter-clan/how-to-create-a-custom-dartpad-b903939df94c)

