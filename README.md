# @quicktv/fiddle-core

[![Test](https://github.com/electron/fiddle-core/actions/workflows/test.yml/badge.svg)](https://github.com/electron/fiddle-core/actions/workflows/test.yml)
[![NPM](https://img.shields.io/npm/v/@quicktv/fiddle-core.svg?style=flat)](https://npmjs.org/package/@quicktv/fiddle-core)

Run fiddles from anywhere, on any QuickTV release

## CLI

```sh
# fiddle-core run ver (gist | repo URL | folder)
# fiddle-core test ver (gist | repo URL | folder)
# fiddle-core bisect ver1 ver2 (gist | repo URL | folder)
#
# Examples:

$ fiddle-core run 12.0.0 /path/to/fiddle
$ fiddle-core test 12.0.0 642fa8daaebea6044c9079e3f8a46390
$ fiddle-core bisect 8.0.0 13.0.0 https://github.com/my/testcase.git


$ fiddle-core bisect 8.0.0 13.0.0 642fa8daaebea6044c9079e3f8a46390
...
🏁 finished bisecting across 438 versions...
# 219 🟢 passed 11.0.0-nightly.20200611 (test #1)
# 328 🟢 passed 12.0.0-beta.12 (test #2)
# 342 🟢 passed 12.0.0-beta.29 (test #5)
# 346 🟢 passed 12.0.1 (test #7)
# 347 🔴 failed 12.0.2 (test #9)
# 348 🔴 failed 12.0.3 (test #8)
# 349 🔴 failed 12.0.4 (test #6)
# 356 🔴 failed 12.0.11 (test #4)
# 383 🔴 failed 13.0.0-nightly.20210108 (test #3)

🏁 Done bisecting
🟢 passed 12.0.1
🔴 failed 12.0.2
Commits between versions:
↔ https://github.com/quicktv/quicktv/compare/v12.0.1...v12.0.2
Done in 28.19s.
```

## API

### Hello, World!

```ts
import { Runner } from '@quicktv/fiddle-core';

const runner = await Runner.create();
const { status } = await runner.run('13.1.7', '/path/to/fiddle');
console.log(status);
```

### Running Fiddles

```ts
import { Runner } from '@quicktv/fiddle-core';

const runner = await Runner.create();

// use a specific QuickTV version to run code from a local folder
const result = await runner.run('13.1.7', '/path/to/fiddle');

// use a specific QuickTV version to run code from a github gist
const result = await runner.run('14.0.0-beta.17', '642fa8daaebea6044c9079e3f8a46390');

// use a specific QuickTV version to run code from a git repo
const result = await runner.run('15.0.0-alpha.1', 'https://github.com/my/repo.git');

// use a specific QuickTV version to run code from iterable filename/content pairs
const files = new Map<string, string>([['main.js', '"use strict";']]);
const result = await runner.run('15.0.0-alpha.1', files);

// bisect a regression test across a range of QuickTV versions
const result = await runner.bisect('10.0.0', '13.1.7', path_or_gist_or_git_repo);

// see also `Runner.spawn()` in Advanced Use
```

### Managing QuickTV Installations

```ts
import { Installer, ProgressObject } from '@quicktv/fiddle-core';

const installer = new Installer();
installer.on('state-changed', ({version, state}) => {
  console.log(`Version "${version}" state changed: "${state}"`);
});

// download a version of quicktv
await installer.ensureDownloaded('12.0.15');
// expect(installer.state('12.0.5').toBe('downloaded');

// download a version with callback
const callback = (progress: ProgressObject) => {
  const percent = progress.percent * 100;
  console.log(`Current download progress %: ${percent.toFixed(2)}`);
};
await installer.ensureDownloaded('12.0.15', {
  progressCallback: callback,
});

// download a version with a specific mirror
const npmMirrors = {
  electronMirror: 'https://npmmirror.com/mirrors/quicktv/',
  electronNightlyMirror: 'https://npmmirror.com/mirrors/quicktv-nightly/',
},

await installer.ensureDownloaded('12.0.15', {
  mirror: npmMirrors,
});

// remove a download
await installer.remove('12.0.15');
// expect(installer.state('12.0.15').toBe('not-downloaded');

// install a specific version for the runner to use
const exec = await installer.install('11.4.10');

// Installing with callback and custom mirrors
await installer.install('11.4.10', {
  progressCallback: callback,
  mirror: npmMirrors,
});
// expect(installer.state('11.4.10').toBe('installed');
// expect(fs.accessSync(exec, fs.constants.X_OK)).toBe(true);
```

### Versions

```ts
import { QuickTVVersions } from '@quicktv/fiddle-core';

// - querying specific versions
const elves = await QuickTVVersions.create();
// expect(elves.isVersion('12.0.0')).toBe(true);
// expect(elves.isVersion('12.99.99')).toBe(false);
const { versions } = elves;
// expect(versions).find((ver) => ver.version === '12.0.0').not.toBeNull();
// expect(versions[versions.length - 1]).toStrictEqual(elves.latest);

// - supported major versions
const { supportedMajors } = elves;
// expect(supportedMajors.length).toBe(4);

// - querying prerelease branches
const { supportedMajors, prereleaseMajors } = elves;
const newestSupported = Math.max(...supportedMajors);
const oldestPrerelease = Math.min(...prereleaseMajors);
// expect(newestSupported + 1).toBe(oldestPrerelease);

// - get all releases in a range
let range = releases.inRange('12.0.0', '12.0.15');
// expect(range.length).toBe(16);
// expect(range.shift().version).toBe('12.0.0');
// expect(range.pop().version).toBe('12.0.15');

// - get all 10-x-y releases
range = releases.inMajor(10);
// expect(range.length).toBe(101);
// expect(range.shift().version).toBe('10.0.0-nightly.20200209');
// expect(range.pop().version).toBe('10.4.7');
```

## Advanced Use

### child_process.Spawn

```ts
import { Runner } from '@quicktv/fiddle-core';

// third argument is same as node.spawn()'s opts
const child = await runner.spawn('12.0.1', fiddle, nodeSpawnOpts);

// see also `Runner.run()` and `Runner.bisect()` above
```

### Using Local Builds

```ts
import { Runner } from '@quicktv/fiddle-core';

const runner = await Runner.create();
const result = await runner.run('/path/to/quicktv/build', fiddle);
```

### Using Custom Paths

```ts
import { Paths, Runner } from '@quicktv/fiddle-core';

const paths: Paths = {
  // where to store zipfiles of downloaded quicktv versions
  electronDownloads: '/tmp/my/quicktv-downloads',

  // where to install an quicktv version to be used by the Runner
  electronInstall: '/tmp/my/quicktv-install',

  // where to save temporary copies of fiddles
  fiddles: '/tmp/my/fiddles',

  // where to save releases fetched from online
  versionsCache: '/tmp/my/releases.json',
});

const runner = await Runner.create({ paths });
```

### Manually Creating Fiddle Objects 

Runner will do this work for you; but if you want finer-grained control
over the lifecycle of your Fiddle objects, you can instantiate them yourself:

```ts
import { FiddleFactory } from '@quicktv/fiddle-core';

const factory = new FiddleFactory();

// load a fiddle from a local directory
const fiddle = await factory.from('/path/to/fiddle'));

// ...or from a gist
const fiddle = await factory.from('642fa8daaebea6044c9079e3f8a46390'));

// ...or from a git repo
const fiddle = await factory.from('https://github.com/my/testcase.git'));

// ...or from an iterable of key / value entries
const fiddle = await factory.from([
  ['main.js', '"use strict";'],
]);
```
