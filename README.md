# 公司的私有vscode插件，如何实现自动更新

我的原文: <https://juejin.cn/post/7141662937420136479/#heading-11>

相关依赖(也是我的项目)：[uvec](https://www.npmjs.com/package/uvec)、[update-vscode-extension](https://www.npmjs.com/package/update-vscode-extension)

## 1. 准备工作

假设你已经有个现成的vscode插件了，如果没有，可以用 `yo generator-code` 创建一个。
然后安装依赖包:

`npm i update-vscode-extension -S`

`npm i uvec -D`

## 2. 编写自动更新逻辑

`update-vscode-extension` 支持对检测更新的自动、手动、暂停、继续等操作。
这里是自动更新的写法, 直接在插件入口 `src/extension.ts` 下注册即可:

```typescript
import vscode, { ExtensionContext } from 'vscode';
import registerUpdateVscodeExtension from 'update-vscode-extension';
import packageJSON from '../package.json';

const registerUpdate = async () => {  
  const { runSlice } = registerUpdateVscodeExtension(packageJSON.name, {
    currentVersion: packageJSON.version,
    vscodeAppRoot: vscode.env.appRoot,
    interval: 10 * 60 * 1000, // 十分钟检测更新一次
  });

  await runSlice();
};

export async function activate(context: ExtensionContext) {
    registerUpdate();
    
    // you code
}
```

或者你想要支持手动更新、并可以支持配置间隔, 也可以实现, 具体的不多说, 大致看看就行:
```typescript
import { isHupuOnline } from '@/utils/utils';
import vscode, { window } from 'vscode';
import registerUpdateVscodeExtension from 'update-vscode-extension';
import packageJSON from '../package.json';

let checkAndUpdate: (() => Promise<void>) | null = null;
let singletonStop: (() => void) | null = null;

const defaultInterval = 10 * 60 * 1000;
const minInterval = 60 * 1000;

const releaseNPMName = '@hupu/vscode-extension';

const statusBarItem = vscode.window.createStatusBarItem(vscode.StatusBarAlignment.Right, 0);

const restartAutoUpdate = (interval: number | null) => {
  singletonStop?.();

  if (interval !== null && interval < minInterval) {
    window.showInformationMessage(`自动更新间隔设置得太小: ${interval}; 已帮你自动设置成: ${defaultInterval}`);
    interval = defaultInterval;
  }

  const { runSlice, stop } = registerUpdateVscodeExtension(releaseNPMName, {
    npmTag: 'latest',
    registryUrl: 'http://hnpm.hupu.io/',
    currentVersion: packageJSON.version,
    vscodeAppRoot: vscode.env.appRoot,
    interval,
    async beforeCheck() {
      if (!await isHupuOnline({ timeout: 5000, checkUrl: 'http://hnpm.hupu.io' })) {
        throw new Error('连不上<http://hnpm.hupu.io>，自动更新插件停止');
      }
    },
    async beforeUpdate(err) {
      if (err) {
        const errMsg = err.toString().includes('404') ? `npm包[${releaseNPMName}]未找到` : err.toString();
        vscode.window.showErrorMessage(`检测插件[键盘侠]最新版本失败: ${errMsg}`);
      } else {
        statusBarItem.text = '插件[键盘侠]自动更新中...';
        statusBarItem.show();
      }
    },
    async afterUpdate(err) {
      statusBarItem.hide();
      if (err) {
        vscode.window.showErrorMessage(`插件自动更新失败: ${err}`);
      } else {
        const buttonLabel = await vscode.window.showInformationMessage(
          `插件[键盘侠]自动更新完毕, 是否重启该窗口`,
          '是',
          '否',
        );
        if (buttonLabel === '是') {
          vscode.commands.executeCommand('workbench.action.reloadWindow');
        }
      }
    },
  });

  checkAndUpdate = runSlice;
  singletonStop = stop;

  runSlice();
};

const registerUpdate = async (ctx: vscode.ExtensionContext) => {
  restartAutoUpdate(null);

  ctx.subscriptions.push(
    statusBarItem,
    vscode.commands.registerCommand('hupu.check-version-and-update'.checkVersionAndUpdate, async () => {
      await checkAndUpdate?.();
    }),
    vscode.commands.registerCommand('hupu.restart-auto-update', async () => {
      const interval = await window.showInputBox({
        title: '设置检查更新的间隔',
        placeHolder: '请设置检查更新的间隔 (单位ms)',
        value: (config.autoUpdateInterval && config.autoUpdateInterval > minInterval) ? `${config.autoUpdateInterval}` : `${defaultInterval}`,
        validateInput(value) {
          if (!value.match(/^\d+$/)) {
            return '必须为数值';
          }
          if (+value < minInterval) {
            return '间隔太小容易卡死...';
          }
          return null;
        },
      });
      if (interval) {
        config.autoUpdateInterval = +interval;
        restartAutoUpdate(+interval);
        window.showInformationMessage('自动更新已开启/重启');
      }
    }),
    vscode.commands.registerCommand('hupu.close-auto-update', async () => {
      singletonStop?.();
      window.showInformationMessage('自动更新已关闭');
    }),
  );
};

export default registerUpdate;
```

## 3. 发布新版本

在package.json中的scripts加入update
```typescript
{
  "scripts": {
    "update": "uvec package . --registry-url='http://hnpm.hupu.io/' --pkg-name='@hupu/vscode-extension' --vsce.no-yarn --vsce.allow-star-activation"
  }
}
```

这里的 `@hupu/vscode-extension` 记得替换成你最终要发布到npm私有仓库上的包名, 另外如果需要加一些vsce运行时的参数，可以通过 `--vece.param=xxx` 的方式进行配置。更详细的可以看 <https://www.npmjs.com/package/uvce>

如果想更新最新版本, 运行 `npm run update` 就行了。

然后结合之前在vscode中写的自动更新逻辑, 你的vscode插件就可以自动更新了。

## 4. 编写 README

先用 `vsce package` 命令打包个vsix文件, 然后上传到cdn上, 让团队的所有人都安装一次, 后面就全靠自动更新了, 不需要再手动安装。

```markdown
## 下载与安装

[点击下载](https://activity-static.hoopchina.com.cn/images/22725-88sd84rc-upload-1658735441279-2.vsix)

安装方式: <https://xxxxxxxx>

安装后在vscode中插件会自动更新到最新版本
```

