# How to build

Please refer to [official Chromium build documentation](https://www.chromium.org/developers/how-tos/get-the-code) to get started on how to build Chromium.

The Chromium version tag used as base for the patches is available here: [RELEASE](../build/RELEASE); this is corresponding to the git tag for every release.
The GN args used to build Cromite are available here: [cromite.gn_args](../build/cromite.gn_args).
The patches are to be applied second the order specified in the `cromite_patches_list.txt` file (you can use `git am`).

A complete, ready-to-build docker image is also available.
Each release contains the description reference containing the name of the corresponding docker image. All images are in the format:

`uazo/cromite-build:(VERSION)-(COMMIT)`

You can see all the images released in https://hub.docker.com/r/uazo/cromite-build/tags

This is the list of commands to perform a build:

Given:
```
SHA=fac696b3a422f196f698e6543913946ddaba1ef3
VERSION=122.0.6261.111
CHVERSION=uazo/cromite-build:$VERSION-$SHA
CONTAINER=dev1
BDEBUG=false
```

execute:

```
docker create --name $CONTAINER \
    -e "WORKSPACE=/home/lg/working_dir" \
	-e "TARGET_ISDEBUG=$BDEBUG" \
    --entrypoint "tail" $CHVERSION "-f" "/dev/null"
docker start dev1
docker exec -ti dev1 bash
```

and, inside the container:
```
PATH=$WORKSPACE/chromium/src/third_party/llvm-build/Release+Asserts/bin:$WORKSPACE/depot_tools/:/usr/local/go/bin:$WORKSPACE/mtool/bin:$PATH
export HOME=/home/lg/working_dir
cd $HOME
cd chromium/src/
```

please note that these commands are valid for the build of android and linux platforms.

for the windows build you need to configure the cross build mode from linux.
You can use the [script](https://github.com/uazo/cromite/blob/master/tools/images/win-sdk/prepare.sh) to generate it. 

all release builds are done via the [action available](https://github.com/uazo/cromite/blob/master/.github/workflows/build_cromite.yaml) from which you can see the mode I have adopted.

Example building process:


```bash
cat build/RELEASE      # this must match docker img version eg. 147.0.7727.102

docker create --name cromite-dev \
    -v "$PWD:/work" \
    -w /work \
    --entrypoint tail \
    uazo/cromite-build:147.0.7727.56-271900671db643de04aa9f909f0dcc3415c8b827 \    # docker img version
    -f /dev/null

docker start cromite-dev
docker exec -it cromite-dev bash

# (Inside container)
export HOME=/home/lg/working_dir
export WORKSPACE=/home/lg/working_dir
PATH=$WORKSPACE/chromium/src/third_party/llvm-build/Release+Asserts/bin:$WORKSPACE/depot_tools/:/usr/local/go/bin:$WORKSPACE/mtool/bin:$PATH
cd $HOME
cd chromium/src/

# clean build (upstream cromite)
# gn gen --args="target_os = \"android\" $(cat /home/lg/working_dir/cromite/build/cromite.gn_args) target_cpu = \"arm64\" " out/arm64

# build with changes (this fork)
gn gen --args="target_os = \"android\" $(cat /work/build/cromite.gn_args) target_cpu = \"arm64\" " out/arm64

unset HTTP_PROXY HTTPS_PROXY http_proxy https_proxy

vpython3 /home/lg/working_dir/depot_tools/siso.py ninja -C out/arm64 chrome_public_apk --offline
cp -r out/arm64/apks/ /work/out
```