---
layout: post
title:  "Pushing to Tidbyte with Github Actions (and how to resolve the libwebp.so.6: cannot open shared object file error)"
author: david
categories: [ Code ]
tags: [ "Github Actions", "Tidbyt", "Pets", "Troubleshooting" ]
image: https://lh3.googleusercontent.com/FYmcs4M67TqoZuRGas2L4paKFyaWhygwds3wReQiTtoFS0WyMVHDu0WWy8buGvbTAf4ZtiLJ1yEs7t562W1jARf_Ly2th1ltt3ZKfbebA5Q9UNGBRdaOMZfui_zHtYPi7Wcxti85mMMa3Z4fWks6p7T8p-c45ZviOELX7JxazRFfdFQfVswr42TXaUW7_bNCtrMAxwgAoWl2wKLldS-vBkMq32IEGkGAyQdtc9fjR0NEfcOewXa-14jFo-CNbhUvzhFrxmu6aptzpw0tgnObJsr_ybPtkLRWDQVaS_kX1VWi8POZB2aikpViVL81W-hNCSVVvHhAcYyTpNQLFo0CiiH5vq2HdlsFNCFA7Q13nGN8PxmuuPVzSez-hzFtSkcZYwQSF6MMV02nI-FSFat9J9hcEt4uKnQmwsGVrnt825-hc44DFOYdLDnAZu31jM0d7_IWLLsi2wlSu-qmk4cioB-opM5Ec3VkPSOm2P7hq0i6NWpi7NFaVViCdhbQ9fvDrNWEPAGI0xPUtGJmu_oR8iJDos3PfjJbtL4r3qCWJR3uuaVhlgsqNxkp1BhfeO9-_-PT5l_9LOShmM26C1xk1GZ2IGRsEUbZ68V16euyc0zUWHh6D7WIRXXNuzszbznBSwnl58ksE9Qr1LkUo5PnL5IxyO4RenpDaWDChdlU1rNkOayz6R4grmx2f-7IYpJOCfoLETcEWboEwe7BuHzeZgf2sqyF3RgoMtfB0g_9UlLRRjdi27cz5cSdWp5XSN91mIxnX3jJCzevfIVKy-OXeBchakDg_-rCJSMe9XejVmZsstsOmx6Vh5FrEdc3uEtwcI-g9C2j8GHpih_YboQGFLPpJXFGB81Wx5ipA4REyKz-bgj7o80QmocDw-XLYR3hBkvXnu5c0URy-qWC2tZV0aJLFrWda0advlg113H5qEJODoX-RxiS7YM6MU6dRRm7mN4-vwFvICl22xbd5FMxcrIvY7QtRnfi2U2jagIDifciMzOeIdXiYus31WbNRMxUaMVtNXVcO-fjWpceTB3CfkrSaQ=w1856-h1392-no
---

## [Click here to skip to the working file](#github-action-with-fixes).

We track the step goal of our pets using a [Fi Collar](https://tryfi.com/) worn by each of them. A Github Action pulls the step data from Fi's API and then renders an image to our [Tidbyt](https://tidbyt.com/), displaying the information (and other things) in our living room.

The result looks something like what you see above (depending on how they're doing with their step goal).

Here's the workflow file:

```yaml
on: 
  workflow_dispatch:
  schedule:
    - cron: '0 * * * *'

jobs:
  render_webp:
    runs-on: ubuntu-latest
    steps:
      - name: Check out current repo
        uses: actions/checkout@v2

      - name: Install Dependencies
        run: sudo apt-get install libwebp-dev -y

      - name: Download pixlet
        uses: dsaltares/fetch-gh-release-asset@master
        with:
          repo: 'tidbyt/pixlet'
          version: 'tags/v0.17.12'
          file: 'pixlet_0.17.12_linux_amd64.tar.gz'

      - name: Unarchive pixlet
        run: tar xzvf pixlet_0.17.12_linux_amd64.tar.gz && chmod +x pixlet

      - name: Render webp Image
        run: |
          ./pixlet render winter_steps.star

      - name: Push webp Image
        run: |
          ./pixlet push \
            --api-token "${{ secrets.TIDBYT_API_TOKEN }}" \
            --installation-id wintersteps \
            "${{ secrets.TIDBYT_DEVICE_ID }}" \
            winter_steps.webp \
            --background
```

Nothing too crazy.

Anyway, I noticed today that the github action was failing - I was getting email alerts, and the steps weren't updating. Looking through the workflow logs, Tidbyt's `pixlet` binary was having some issues:

```
Run ./pixlet render winter_steps.star
./pixlet: error while loading shared libraries: libwebp.so.6: cannot open shared object file: No such file or directory
```

Weird.  Just a few lines up, I install the libwebp-dev package that's been working up until recently.

```
sudo apt-get install libwebp-dev -y
```

I modified the workflow to install `apt-file`, update its inventory, and then searched for where libwebp-dev was putting it's `.so` files.

```yaml
# [...]
        - name: Install apt-file
        run: sudo apt-get install apt-file -y

        - name: Where does it put it
        run: sudo apt-file update && sudo apt-file list libwebp-dev
# [...]
```

The output looks like:

```sh
Fetched 124 MB in 24s (5079 kB/s)
Run sudo apt-file update && sudo apt-file list libwebp-dev
Get:1 https://packages.microsoft.com/ubuntu/22.04/prod jammy InRelease [10.5 kB]
# [...]
Reading package lists...
libwebp-dev: /usr/include/webp/decode.h
libwebp-dev: /usr/include/webp/demux.h
libwebp-dev: /usr/include/webp/encode.h
# [...]
libwebp-dev: /usr/lib/x86_64-linux-gnu/libwebp.so
libwebp-dev: /usr/lib/x86_64-linux-gnu/libwebpdemux.so
libwebp-dev: /usr/lib/x86_64-linux-gnu/libwebpmux.so
# [...]
```

Sure enough, it's in a new location, under a different name. A package update must have changed the default filename, or the latest `ubuntu-latest` maybe have relocated the library path away from where `pixlet` was expecting it. Either way, this is not the `libwebp.so.6` we expected.

So what's the fix?

`pixlet` is looking for `libwebp.so.6` - let's give it `libwebp.so.6`.

```yaml
- name: Put libwebp where it's expected
run: sudo ln /usr/lib/x86_64-linux-gnu/libwebp.so /usr/lib/x86_64-linux-gnu/libwebp.so.6
```

And let's make sure it's looking in the right directory for these lib files.

```yaml
- name: Render webp Image
run: ./pixlet render winter_steps.star
env: 
    LD_LIBRARY_PATH: /usr/lib/x86_64-linux-gnu
```

And now we're done! Here's the full github action for anyone interested. Hope it helps someone.

![](/assets/images/post_images/winter_steps.png)

### Github Action with Fixes

```yaml
on: 
  workflow_dispatch:
  schedule:
    - cron: '0 * * * *'

jobs:
  render_webp:
    runs-on: ubuntu-latest
    steps:
      - name: Check out current repo
        uses: actions/checkout@v2

      - name: Install Dependencies
        run: sudo apt-get install libwebp-dev -y

        # We added this!
      - name: Put libwebp where it's expected
        run: sudo ln /usr/lib/x86_64-linux-gnu/libwebp.so /usr/lib/x86_64-linux-gnu/libwebp.so.6

      - name: Download pixlet
        uses: dsaltares/fetch-gh-release-asset@master
        with:
          repo: 'tidbyt/pixlet'
          version: 'tags/v0.17.12'
          file: 'pixlet_0.17.12_linux_amd64.tar.gz'

      - name: Unarchive pixlet
        run: tar xzvf pixlet_0.17.12_linux_amd64.tar.gz && chmod +x pixlet

      - name: Render webp Image
        run: ./pixlet render winter_steps.star
        # We added the `env` field here
        env: 
          LD_LIBRARY_PATH: /usr/lib/x86_64-linux-gnu

      - name: Push webp Images
        run: |
            --api-token "${{ secrets.TIDBYT_API_TOKEN }}" \
            --installation-id wintersteps \
            "${{ secrets.TIDBYT_DEVICE_ID }}" \
            winter_steps.webp \
            --background
```