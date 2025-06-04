# Cocos Creator Hot Update Remote Shell Tutorial

This guide shows you how to create a Cocos Creator mobile app that can remotely update its UI and logic after release, using the hot update system. The app starts as a simple shell (e.g., a label), and after a hot update, can show new content (like an image screen) from the server.

---

## Step 1: Create the Minimal Shell App

1. In Cocos Creator, create a new project.
2. In your main scene, add a **Canvas** with a **GameContent** node and a **Label** child. Set the label text to "Hello World".
3. Create a script called `RemoteEntryLoader.ts` and attach it to the Canvas (or a root node).
4. In the Inspector, assign the **GameContent** node to the `gameContent` property of `RemoteEntryLoader`.

### Sample `RemoteEntryLoader.ts`
```typescript
import { _decorator, Component, assetManager, Prefab, instantiate, JsonAsset, Node } from 'cc';
const { ccclass, property } = _decorator;

@ccclass('RemoteEntryLoader')
export class RemoteEntryLoader extends Component {
    @property(Node)
    gameContent: Node = null;

    onLoad() {
        // After hot update, always load the entry config from the remote bundle
        assetManager.loadBundle('config', (err, bundle) => {
            if (err) {
                // Optionally, try loading from remote URL if not found locally
                assetManager.loadBundle('https://abc.com/config', (err2, bundle2) => {
                    if (err2) {
                        console.error('Failed to load config bundle:', err2);
                        return;
                    }
                    this.loadEntry(bundle2);
                });
                return;
            }
            this.loadEntry(bundle);
        });
    }

    private loadEntry(bundle) {
        bundle.load('entry', JsonAsset, (err, jsonAsset) => {
            if (err) {
                console.error('Failed to load entry config:', err);
                return;
            }
            const entry = jsonAsset.json; // e.g., { "mainPrefab": "ImageScreen" }
            bundle.load(entry.mainPrefab, Prefab, (err, prefab) => {
                if (err) {
                    console.error('Failed to load main prefab:', err);
                    return;
                }
                // Remove the old GameContent label if it exists
                if (this.gameContent && this.gameContent.isValid) {
                    this.gameContent.destroy();
                }
                const node = instantiate(prefab);
                node.parent = this.node;
            });
        });
    }
}
```

5. **Do NOT** include a `config` folder or bundle in your assets for the initial build.
6. Build your APK and install it on your mobile device. The app should show only the "Hello World" label.

---

## Before Building the APK: Assets Folder Structure

![Assets before release APK](https://drive.google.com/file/d/1wUiwKqt-YjF77gSBn_c2FQ37iv_6RqQP/view?usp=sharing)

```
assets/
├── scenes/
│   └── scene
├── scripts/
│   └── RemoteEntryLoader.ts
└── internal/
```
- **No `config` folder or bundle is present at this stage.**
- This ensures your initial APK is a true remote shell.

---

## Step 2: Prepare the Remote Bundle (for Hot Update)

1. In Cocos Creator, create a folder in your assets called `config`.
2. In `config`, create a new prefab called `ImageScreen`:
    - Add a **Sprite** node and assign an image to its SpriteFrame.
    - Add a **Label** node as a child of `ImageScreen`.
    - (Optional) Create a script `ImageScreenLogic.ts` in `config` and attach it to `ImageScreen` to set the label text.

### Sample `ImageScreenLogic.ts` (for remote logic)
```typescript
import { _decorator, Component, Label } from 'cc';
const { ccclass, property } = _decorator;

@ccclass('ImageScreenLogic')
export class ImageScreenLogic extends Component {
    @property(Label)
    label: Label = null;

    start() {
        if (this.label) {
            this.label.string = "This text is set by remote logic!";
        }
    }
}
```

3. In `config`, create `entry.json` with:
```json
{
  "mainPrefab": "ImageScreen"
}
```
4. **Mark the `config` folder as an Asset Bundle:**
    - Select the `config` folder in the Assets panel.
    - In the Inspector, check **Is Bundle**, set **Bundle Name** to `config` (see screenshot below).

![Inspector: Mark config as bundle](https://drive.google.com/file/d/1GlnD9kY5BjsBfghl8CF7Q2RTmqD_Sg03/view?usp=sharing)

---

## After Building the APK: Assets Folder Structure

![Assets after release first APK](https://drive.google.com/file/d/1wUiwKqt-YjF77gSBn_c2FQ37iv_6RqQP/view?usp=sharing)
[Imgur](https://imgur.com/NaBclWP)

```
assets/
├── config/   (marked as bundle)
│   ├── entry.json
│   ├── image/(your images)
│   │── ImageScreen.prefab
│   │── ImageScreenLogic.ts
│   │── ...
├── scenes/
│   └── scene
├── scripts/
│   └── RemoteEntryLoader.ts
└── internal/
```
- **The `config` folder is now present and marked as a bundle.**
- This bundle can now be built and updated remotely for hot update.

---

## Step 3: Build the Bundle and Generate Manifests

1. Build your project for Android/iOS. The bundle will be generated in your build output (e.g., `build/android/assets/assets/config/`).
2. Use `version_generator.js` to generate `project.manifest` and `version.manifest` for the bundle.  
   Example command:
   ```sh
   node version_generator.js -v 1.0.0 -u https://YOUR_SERVER_URL/config/ -s PATH_TO_YOUR_BUNDLE -d PATH_TO_YOUR_BUNDLE
   ```
   **Note:** Make sure your terminal is in the same directory as `version_generator.js` when running this command, or provide the correct path to the script.

---

## Download `version_generator.js`

The `version_generator.js` script is used to generate the `project.manifest` and `version.manifest` files for your hot update bundle. These manifest files are required for the Cocos Creator hot update system to work correctly.


---

## Step 4: Upload to Server

1. Upload the entire built `config` folder (including all files, prefabs, images, and manifests) to your static host (e.g., GitHub Pages, S3, etc.).

---

## Step 5: Test the Hot Update

1. Launch your app on your device. On first launch, it will show only the "Hello World" label.
2. After you upload the `config` bundle to your server, relaunch the app.
3. The app will hot update, load the new prefab, and you will see the image and label from the remote bundle.

---

## Tips & Notes

- For native (mobile) apps, you can also hot update scripts (as compiled JS) in the remote bundle.
- For web builds, all scripts must be present in the initial build.
- Always build the bundle and generate new manifests after adding or changing files in `config`.
- Make sure to upload the entire built bundle, including the `import/` folder and all assets. 
