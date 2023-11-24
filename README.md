# Azarbayjab1-
name: Release Build
on:
  workflow_dispatch:
    inputs:
      tag:
        description: 'Release Tag'
        required: true
      upload:
        description: 'Upload: If want ignore'
        required: false
      publish:
        description: 'Publish: If want ignore'
        required: false
      play:
        description: 'Play: If want ignore'
        required: false
jobs:
  libcore:
    name: Native Build (LibCore)
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Golang Status
        run: find buildScript libcore/*.sh | xargs cat | sha1sum > golang_status
      - name: Libcore Status
        run: git ls-files libcore | xargs cat | sha1sum > libcore_status
      - name: LibCore Cache
        id: cache
        uses: actions/cache@v3
        with:
          path: |
            app/libs/libcore.aar
          key: ${{ hashFiles('.github/workflows/*', 'golang_status', 'libcore_status') }}
      - name: Golang Cache
        if: steps.cache.outputs.cache-hit != 'true'
        uses: actions/cache@v3
        with:
          path: build/golang
          key: go-${{ hashFiles('.github/workflows/*', 'golang_status') }}
      - name: Native Build
        if: steps.cache.outputs.cache-hit != 'true'
        run: ./run init action go && ./run lib core
  build:
    name: Build OSS APK
    runs-on: ubuntu-latest
    needs:
      - libcore
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Golang Status
        run: find buildScript libcore/*.sh | xargs cat | sha1sum > golang_status
      - name: Libcore Status
        run: git ls-files libcore | xargs cat | sha1sum > libcore_status
      - name: LibCore Cache
        uses: actions/cache@v3
        with:
          path: |
            app/libs/libcore.aar
          key: ${{ hashFiles('.github/workflows/*', 'golang_status', 'libcore_status') }}
      - name: Gradle cache
        uses: actions/cache@v3
        with:
          path: ~/.gradle
          key: gradle-oss-${{ hashFiles('**/*.gradle.kts') }}
      - name: Gradle Build
        env:
          BUILD_PLUGIN: none
        run: |
          echo "sdk.dir=${ANDROID_HOME}" > local.properties
          echo "ndk.dir=${ANDROID_HOME}/ndk/25.0.8775105" >> local.properties
          export LOCAL_PROPERTIES="${{ secrets.LOCAL_PROPERTIES }}"
          ./run init action gradle
          ./gradlew app:assembleOssRelease
          APK=$(find app/build/outputs/apk -name '*arm64-v8a*.apk')
          APK=$(dirname $APK)
          echo "APK=$APK" >> $GITHUB_ENV
      - uses: actions/upload-artifact@v3
        with:
          name: APKs
          path: ${{ env.APK }}
  publish:
    name: Publish Release
    if: github.event.inputs.publish != 'y'
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Donwload Artifacts
        uses: actions/download-artifact@v3
        with:
          name: APKs
          path: artifacts
      - name: Release
        run: |
          wget -O ghr.tar.gz https://github.com/tcnksm/ghr/releases/download/v0.13.0/ghr_v0.13.0_linux_amd64.tar.gz
          tar -xvf ghr.tar.gz
          mv ghr*linux_amd64/ghr .
          mkdir apks
          find artifacts -name "*.apk" -exec cp {} apks \;
          ./ghr -delete -t "${{ github.token }}" -n "${{ github.event.inputs.tag }}" "${{ github.event.inputs.tag }}" apks
  upload:
    name: Upload Release
    if: github.event.inputs.upload != 'y'
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Donwload Artifacts
        uses: actions/download-artifact@v3
        with:
          name: APKs
          path: artifacts
      - name: Release
        run: |
          mkdir apks
          find artifacts -name "*.apk" -exec cp {} apks \;

          function upload() {
            for apk in $@; do
              echo ">> Uploading $apk"
              curl https://api.telegram.org/bot${{ secrets.TELEGRAM_TOKEN }}/sendDocument \
                -X POST \
                -F chat_id="${{ secrets.TELEGRAM_CHANNEL }}" \
                -F document="@$apk" \
                --silent --show-error --fail >/dev/null &
            done
            for job in $(jobs -p); do
              wait $job || exit 1
            done
          }
          upload apks/*
  play:
    name: Build Play Bundle
    if: github.event.inputs.play != 'y'
    runs-on: ubuntu-latest
    needs:
      - libcore
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Golang Status
        run: find buildScript libcore/*.sh | xargs cat | sha1sum > golang_status
      - name: Libcore Status
        run: git ls-files libcore | xargs cat | sha1sum > libcore_status
      - name: LibCore Cache
        uses: actions/cache@v3
        with:
          path: |
            app/libs/libcore.aar
          key: ${{ hashFiles('.github/workflows/*', 'golang_status', 'libcore_status') }}
      - name: Gradle cache
        uses: actions/cache@v3
        with:
          path: ~/.gradle
          key: gradle-play-${{ hashFiles('**/*.gradle.kts') }}
      - name: Checkout Library
        run: |
          git submodule update --init 'app/*'
      - name: Gradle Build
        run: |
          echo "sdk.dir=${ANDROID_HOME}" > local.properties
          echo "ndk.dir=${ANDROID_HOME}/ndk/25.0.8775105" >> local.properties
          export LOCAL_PROPERTIES="${{ secrets.LOCAL_PROPERTIES }}"
          ./run init action gradle
          ./gradlew bundlePlayRelease
      - uses: actions/upload-artifact@v3
        with:
          name: AAB
          path: app/build/outputs/bundle/playRelease/app-play-release.aab 

]
]
name: Release Build
on:
  workflow_dispatch:
    inputs:
      tag:
        description: 'Release Tag'
        required: true
      upload:
        description: 'Upload: If want ignore'
        required: false
      publish:
        description: 'Publish: If want ignore'
        required: false
      play:
        description: 'Play: If want ignore'
        required: false
jobs:
  libcore:
    name: Native Build (LibCore)
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Golang Status
        run: find buildScript libcore/*.sh | xargs cat | sha1sum > golang_status
      - name: Libcore Status
        run: git ls-files libcore | xargs cat | sha1sum > libcore_status
      - name: LibCore Cache
        id: cache
        uses: actions/cache@v3
        with:
          path: |
            app/libs/libcore.aar
          key: ${{ hashFiles('.github/workflows/*', 'golang_status', 'libcore_status') }}
      - name: Golang Cache
        if: steps.cache.outputs.cache-hit != 'true'
        uses: actions/cache@v3
        with:
          path: build/golang
          key: go-${{ hashFiles('.github/workflows/*', 'golang_status') }}
      - name: Native Build
        if: steps.cache.outputs.cache-hit != 'true'
        run: ./run init action go && ./run lib core
  build:
    name: Build OSS APK
    runs-on: ubuntu-latest
    needs:
      - libcore
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Golang Status
        run: find buildScript libcore/*.sh | xargs cat | sha1sum > golang_status
      - name: Libcore Status
        run: git ls-files libcore | xargs cat | sha1sum > libcore_status
      - name: LibCore Cache
        uses: actions/cache@v3
        with:
          path: |
            app/libs/libcore.aar
          key: ${{ hashFiles('.github/workflows/*', 'golang_status', 'libcore_status') }}
      - name: Gradle cache
        uses: actions/cache@v3
        with:
          path: ~/.gradle
          key: gradle-oss-${{ hashFiles('**/*.gradle.kts') }}
      - name: Gradle Build
        env:
          BUILD_PLUGIN: none
        run: |
          echo "sdk.dir=${ANDROID_HOME}" > local.properties
          echo "ndk.dir=${ANDROID_HOME}/ndk/25.0.8775105" >> local.properties
          export LOCAL_PROPERTIES="${{ secrets.LOCAL_PROPERTIES }}"
          ./run init action gradle
          ./gradlew app:assembleOssRelease
          APK=$(find app/build/outputs/apk -name '*arm64-v8a*.apk')
          APK=$(dirname $APK)
          echo "APK=$APK" >> $GITHUB_ENV
      - uses: actions/upload-artifact@v3
        with:
          name: APKs
          path: ${{ env.APK }}
  publish:
    name: Publish Release
    if: github.event.inputs.publish != 'y'
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Donwload Artifacts
        uses: actions/download-artifact@v3
        with:
          name: APKs
          path: artifacts
      - name: Release
        run: |
          wget -O ghr.tar.gz https://github.com/tcnksm/ghr/releases/download/v0.13.0/ghr_v0.13.0_linux_amd64.tar.gz
          tar -xvf ghr.tar.gz
          mv ghr*linux_amd64/ghr .
          mkdir apks
          find artifacts -name "*.apk" -exec cp {} apks \;
          ./ghr -delete -t "${{ github.token }}" -n "${{ github.event.inputs.tag }}" "${{ github.event.inputs.tag }}" apks
  upload:
    name: Upload Release
    if: github.event.inputs.upload != 'y'
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Donwload Artifacts
        uses: actions/download-artifact@v3
        with:
          name: APKs
          path: artifacts
      - name: Release
        run: |
          mkdir apks
          find artifacts -name "*.apk" -exec cp {} apks \;

          function upload() {
            for apk in $@; do
              echo ">> Uploading $apk"
              curl https://api.telegram.org/bot${{ secrets.TELEGRAM_TOKEN }}/sendDocument \
                -X POST \
                -F chat_id="${{ secrets.TELEGRAM_CHANNEL }}" \
                -F document="@$apk" \
                --silent --show-error --fail >/dev/null &
            done
            for job in $(jobs -p); do
              wait $job || exit 1
            done
          }
          upload apks/*
  play:
    name: Build Play Bundle
    if: github.event.inputs.play != 'y'
    runs-on: ubuntu-latest
    needs:
      - libcore
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Golang Status
        run: find buildScript libcore/*.sh | xargs cat | sha1sum > golang_status
      - name: Libcore Status
        run: git ls-files libcore | xargs cat | sha1sum > libcore_status
      - name: LibCore Cache
        uses: actions/cache@v3
        with:
          path: |
            app/libs/libcore.aar
          key: ${{ hashFiles('.github/workflows/*', 'golang_status', 'libcore_status') }}
      - name: Gradle cache
        uses: actions/cache@v3
        with:
          path: ~/.gradle
          key: gradle-play-${{ hashFiles('**/*.gradle.kts') }}
      - name: Checkout Library
        run: |
          git submodule update --init 'app/*'
      - name: Gradle Build
        run: |
          echo "sdk.dir=${ANDROID_HOME}" > local.properties
          echo "ndk.dir=${ANDROID_HOME}/ndk/25.0.8775105" >> local.properties
          export LOCAL_PROPERTIES="${{ secrets.LOCAL_PROPERTIES }}"
          ./run init action gradle
          ./gradlew bundlePlayRelease
      - uses: actions/upload-artifact@v3
        with:
          name: AAB
          path: app/build/outputs/bundle/playRelease/app-play-release.aabinstagram.video.downloader.story.saver
com.sdev.alphav2ray
com.binwizteam.vpn
com.microsoft.emmx
com.sec.android.app.samsungapps
com.nexters.herowars
com.instagram.android
live.wallpaper.widgets3d
ir.mci.ecareapp
com.netmod.syna
com.sec.android.app.billing
com.v2cross.proxy
com.game.space.shooter2
com.spotify.music
com.instagram.barcelona
com.zhiliaoapp.musically
com.transparent.livewallpaper.screen.transparentwallpaper.digitalclock
com.crosserr.trojan
com.agn.v2ray
cn.wps.moffice_eng
com.wave.livewallpaper
video.player.videoplayer
com.v2ray.v2fly
com.v2ray.f
com.v2ray.ang
com.facebook.katana
com.miui.videoplayer