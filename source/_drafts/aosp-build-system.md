---
title: aosp-build-system
tags:
cover:
---





# Android build system启动



基于Android14.0.0_r73



# envsetup.sh启动





## 1.执行获取top路径



这里一般会执行到PWD= /bin/pwd获取当前的路径，并返回给T=$(_gettop_once)

PWD= /bin/pwd会先给PWD设置为空然后执行/bin/pwd将路径输出到output下。

``` shell
function _gettop_once
{
    local TOPFILE=build/make/core/envsetup.mk
    if [ -n "$TOP" -a -f "$TOP/$TOPFILE" ] ; then
        # The following circumlocution ensures we remove symlinks from TOP.
        (cd "$TOP"; PWD= /bin/pwd)
    else
        if [ -f $TOPFILE ] ; then
            # The following circumlocution (repeated below as well) ensures
            # that we record the true directory name and not one that is
            # faked up with symlink names.
            PWD= /bin/pwd
        else
            local HERE=$PWD
            local T=
            while [ \( ! \( -f $TOPFILE \) \) -a \( "$PWD" != "/" \) ]; do
                \cd ..
                T=`PWD= /bin/pwd -P`
            done
            \cd "$HERE"
            if [ -f "$T/$TOPFILE" ]; then
                echo "$T"
            fi
        fi
    fi
}
T=$(_gettop_once)
```



## 2.执行外部脚本 shell_utils



好像只是一个比较小的工具类。设置了3个函数，没啥别的用处。
感觉功能还有些重复...可能是历史遗留代码。



这里的T在#1时已经设置为了工程的top路径。也就是aosp源码的根路径

``` shell
IMPORTING_ENVSETUP=true source $T/build/make/shell_utils.sh
```



关于shell_utils代码逻辑很简单

```shell
# 获取top工程路径，类似于_gettop_once
function gettop
{
....
}

# Sets TOP, or if the root of the tree can't be found, prints a message and
# exits.  Since this function exits, it should not be called from functions
# defined in envsetup.sh.
# IMPORTING_ENVSETUP如果未定义，则定义一个函数用于设置top的值
if [ -z "${IMPORTING_ENVSETUP:-}" ] ; then
function require_top
{
    TOP=$(gettop)
    if [[ ! $TOP ]] ; then
        echo "Can not locate root of source tree. $(basename $0) must be run from within the Android source tree." >&2
        exit 1
    fi
}
fi

# 获取输出路径
function getoutdir
{
    local top=$(gettop)
    local out_dir="${OUT_DIR:-}"
    if [[ -z "${out_dir}" ]]; then
        if [[ -n "${OUT_DIR_COMMON_BASE:-}" && -n "${top}" ]]; then
            out_dir="${OUT_DIR_COMMON_BASE}/$(basename ${top})"
        else
            out_dir="out"
        fi
    fi
    if [[ "${out_dir}" != /* ]]; then
        out_dir="${top}/${out_dir}"
    fi
    echo "${out_dir}"
}
```



## 3.定义 VARIANT_CHOICES



定义了一个字典，值为user or userdebug or eng

```shell
VARIANT_CHOICES=(user userdebug eng)
```



## 4. unset变量



```shell
unset COMMON_LUNCH_CHOICES_CACHE
```





## 5.validate_current_shell

教研shell是bash/zsh. （不支持其他shell）

```shell
function validate_current_shell() {
    local current_sh="$(ps -o command -p $$)"
    case "$current_sh" in
        *bash*)
            function check_type() { type -t "$1"; }
            ;;
        *zsh*)
            function check_type() { type "$1"; }
            enable_zsh_completion ;;
        *)
            echo -e "WARNING: Only bash and zsh are supported.\nUse of other shell would lead to erroneous results."
            ;;
    esac
}

validate_current_shell

```



## 6.set_global_paths



设置一些全局变量

```shell
function set_global_paths()
{
    local T=$(gettop)
    if [ ! "$T" ]; then
        echo "Couldn't locate the top of the tree.  Try setting TOP."
        return
    fi

    ##################################################################
    #                                                                #
    #              Read me before you modify this code               #
    #                                                                #
    #   This function sets ANDROID_GLOBAL_BUILD_PATHS to what it is  #
    #   adding to PATH, and the next time it is run, it removes that #
    #   from PATH.  This is required so envsetup.sh can be sourced   #
    #   more than once and still have working paths.                 #
    #                                                                #
    ##################################################################

    # Out with the old...
    if [ -n "$ANDROID_GLOBAL_BUILD_PATHS" ] ; then
        export PATH=${PATH/$ANDROID_GLOBAL_BUILD_PATHS/}
    fi

    # And in with the new...
    ANDROID_GLOBAL_BUILD_PATHS=$T/build/soong/bin
    ANDROID_GLOBAL_BUILD_PATHS+=:$T/build/bazel/bin
    ANDROID_GLOBAL_BUILD_PATHS+=:$T/development/scripts
    ANDROID_GLOBAL_BUILD_PATHS+=:$T/prebuilts/devtools/tools

    # add kernel specific binaries
    if [ $(uname -s) = Linux ] ; then
        ANDROID_GLOBAL_BUILD_PATHS+=:$T/prebuilts/misc/linux-x86/dtc
        ANDROID_GLOBAL_BUILD_PATHS+=:$T/prebuilts/misc/linux-x86/libufdt
    fi

    # If prebuilts/android-emulator/<system>/ exists, prepend it to our PATH
    # to ensure that the corresponding 'emulator' binaries are used.
    case $(uname -s) in
        Darwin)
            ANDROID_EMULATOR_PREBUILTS=$T/prebuilts/android-emulator/darwin-x86_64
            ;;
        Linux)
            ANDROID_EMULATOR_PREBUILTS=$T/prebuilts/android-emulator/linux-x86_64
            ;;
        *)
            ANDROID_EMULATOR_PREBUILTS=
            ;;
    esac
    if [ -n "$ANDROID_EMULATOR_PREBUILTS" -a -d "$ANDROID_EMULATOR_PREBUILTS" ]; then
        ANDROID_GLOBAL_BUILD_PATHS+=:$ANDROID_EMULATOR_PREBUILTS
        export ANDROID_EMULATOR_PREBUILTS
    fi

    # Finally, set PATH
    export PATH=$ANDROID_GLOBAL_BUILD_PATHS:$PATH
}
```





## 7.source_vendorsetup

在当前路径寻找allowed-vendorsetup_sh-files

vendorsetup.sh并执行

```shell
function source_vendorsetup() {
    unset VENDOR_PYTHONPATH
    local T="$(gettop)"
    allowed=
    for f in $(cd "$T" && find -L device vendor product -maxdepth 4 -name 'allowed-vendorsetup_sh-files' 2>/dev/null | sort); do
        if [ -n "$allowed" ]; then
            echo "More than one 'allowed_vendorsetup_sh-files' file found, not including any vendorsetup.sh files:"
            echo "  $allowed"
            echo "  $f"
            return
        fi
        allowed="$T/$f"
    done

    allowed_files=
    [ -n "$allowed" ] && allowed_files=$(cat "$allowed")
    for dir in device vendor product; do
        for f in $(cd "$T" && test -d $dir && \
            find -L $dir -maxdepth 4 -name 'vendorsetup.sh' 2>/dev/null | sort); do

            if [[ -z "$allowed" || "$allowed_files" =~ $f ]]; then
                echo "including $f"; . "$T/$f"
            else
                echo "ignoring $f, not in $allowed"
            fi
        done
    done

    if [[ "${PWD}" == /google/cog/* ]]; then
        f="build/make/cogsetup.sh"
        echo "including $f"; . "$T/$f"
    fi
}
```







## 8.addcompletions

添加代码补全

```shell
function addcompletions()
{
    local f=

    # Keep us from trying to run in something that's neither bash nor zsh.
    if [ -z "$BASH_VERSION" -a -z "$ZSH_VERSION" ]; then
        return
    fi

    # Keep us from trying to run in bash that's too old.
    if [ -n "$BASH_VERSION" -a ${BASH_VERSINFO[0]} -lt 3 ]; then
        return
    fi

    local completion_files=(
      packages/modules/adb/adb.bash
      system/core/fastboot/fastboot.bash
      tools/asuite/asuite.sh
      prebuilts/bazel/common/bazel-complete.bash
    )
    # Completion can be disabled selectively to allow users to use non-standard completion.
    # e.g.
    # ENVSETUP_NO_COMPLETION=adb # -> disable adb completion
    # ENVSETUP_NO_COMPLETION=adb:bit # -> disable adb and bit completion
    local T=$(gettop)
    for f in ${completion_files[*]}; do
        f="$T/$f"
        if [ ! -f "$f" ]; then
          echo "Warning: completion file $f not found"
        elif should_add_completion "$f"; then
            . $f
        fi
    done

    if should_add_completion bit ; then
        complete -C "bit --tab" bit
    fi
    if [ -z "$ZSH_VERSION" ]; then
        # Doesn't work in zsh.
        complete -o nospace -F _croot croot
        # TODO(b/244559459): Support b autocompletion for zsh
        complete -F _bazel__complete -o nospace b
    fi
    complete -F _lunch lunch

    complete -F _complete_android_module_names pathmod
    complete -F _complete_android_module_names gomod
    complete -F _complete_android_module_names outmod
    complete -F _complete_android_module_names installmod
    complete -F _complete_android_module_names bmod
    complete -F _complete_android_module_names m
}
```























```

validate_current_shell
set_global_paths
source_vendorsetup
addcompletions

```





## 其他函数定义



function _gettop_once
function hmm() 
function build_build_var_cache()
function destroy_build_var_cache()
function get_abs_build_var()
function get_build_var()
function check_product()
function check_variant()
function set_lunch_paths()
function set_global_paths()
function printconfig()
function set_stuff_for_environment()
function set_sequence_number()
function should_add_completion() {
function addcompletions()
function multitree_lunch_help()
function multitree_lunch()
function choosetype()

function chooseproduct()
function choosevariant()
function choosecombo()
function add_lunch_combo()
function print_lunch_menu()
function lunch()
function _lunch()
function tapas()
function banchan()
function multitree_gettop
function croot()
function _croot()
function cproj()
function adb() {
function qpid() {
function syswrite() {
function coredump_setup()
function coredump_enable()
function core()
function systemstack()
function is64bit()
        function sgrep()
        function sgrep()
function gettargetarch
function ggrep()
function gogrep()
function jgrep()
function rsgrep()
function jsongrep()
function tomlgrep()
function ktgrep()
function cgrep()
function resgrep()
function mangrep()
function owngrep()
function sepgrep()
function rcgrep()
function pygrep()
        function mgrep()
        function treegrep()
        function mgrep()
        function treegrep()
function getprebuilt
function runhat()
function getbugreports()
function getsdcardpath()
function getscreenshotpath()
function getlastscreenshot()
function startviewserver()
function stopviewserver()
function isviewserverstarted()
function key_home()
function key_back()
function key_menu()
function smoketest()
function runtest()
function godir () {
function refreshmod() {
function verifymodinfo() {
function allmod() {
function bmod()
function pathmod() {
function dirmods() {
function gomod() {
function outmod() {
function installmod() {
function _complete_android_module_names() {
function pez {
function get_make_command()
function _wrap_build()

function _trigger_build()
function m()
function mm()
function mmm()
function mma()
function mmma()
function make()
function _multitree_lunch_error()
function multitree_build()
function provision()
function enable_zsh_completion() {
function validate_current_shell() {
            function check_type() { type -t "$1"; }
            function check_type() { type "$1"; }
function source_vendorsetup() {
function showcommands() {
function avbtool() {
function overrideflags() {
function aninja() {



# lunch执行



``` shell

function lunch()
{
    local answer
	# 参数传入错误
    if [[ $# -gt 1 ]]; then
        echo "usage: lunch [target]" >&2
        return 1
    fi

    local used_lunch_menu=0

    if [ "$1" ]; then
        answer=$1
    else
        print_lunch_menu
        echo "Which would you like? [aosp_cf_x86_64_phone-trunk_staging-eng]"
        echo -n "Pick from common choices above (e.g. 13) or specify your own (e.g. aosp_barbet-trunk_staging-eng): "
        read answer
        used_lunch_menu=1
    fi

    local selection=

    if [ -z "$answer" ]
    then
        selection=aosp_cf_x86_64_phone-trunk_staging-eng
    elif (echo -n $answer | grep -q -e "^[0-9][0-9]*$")
    then
        local choices=($(TARGET_BUILD_APPS= TARGET_PRODUCT= TARGET_RELEASE= TARGET_BUILD_VARIANT= get_build_var COMMON_LUNCH_CHOICES 2>/dev/null))
        if [ $answer -le ${#choices[@]} ]
        then
            # array in zsh starts from 1 instead of 0.
            if [ -n "$ZSH_VERSION" ]
            then
                selection=${choices[$(($answer))]}
            else
                selection=${choices[$(($answer-1))]}
            fi
        fi
    else
        selection=$answer
    fi

    export TARGET_BUILD_APPS=

    # This must be <product>-<release>-<variant>
    local product release variant
    # Split string on the '-' character.
    IFS="-" read -r product release variant <<< "$selection"

    if [[ -z "$product" ]] || [[ -z "$release" ]] || [[ -z "$variant" ]]
    then
        echo
        echo "Invalid lunch combo: $selection"
        echo "Valid combos must be of the form <product>-<release>-<variant>"
        return 1
    fi

    TARGET_PRODUCT=$product \
    TARGET_BUILD_VARIANT=$variant \
    TARGET_RELEASE=$release \
    build_build_var_cache
    if [ $? -ne 0 ]
    then
        if [[ "$product" =~ .*_(eng|user|userdebug) ]]
        then
            echo "Did you mean -${product/*_/}? (dash instead of underscore)"
        fi
        return 1
    fi
    export TARGET_PRODUCT=$(get_build_var TARGET_PRODUCT)
    export TARGET_BUILD_VARIANT=$(get_build_var TARGET_BUILD_VARIANT)
    export TARGET_RELEASE=$release
    # Note this is the string "release", not the value of the variable.
    export TARGET_BUILD_TYPE=release

    [[ -n "${ANDROID_QUIET_BUILD:-}" ]] || echo

    set_stuff_for_environment
    [[ -n "${ANDROID_QUIET_BUILD:-}" ]] || printconfig

    if [ "${TARGET_BUILD_VARIANT}" = "userdebug" ] && [[  -z "${ANDROID_QUIET_BUILD}" ]]; then
      echo
      echo "Want FASTER LOCAL BUILDS? Use -eng instead of -userdebug (however for" \
        "performance benchmarking continue to use userdebug)"
    fi
    if [ $used_lunch_menu -eq 1 ]; then
      echo
      echo "Hint: next time you can simply run 'lunch $selection'"
    fi

    destroy_build_var_cache

    if [[ -n "${CHECK_MU_CONFIG:-}" ]]; then
      check_mu_config
    fi
}
```



# m



这个方法里面没有什么东西，主要调用了一个方法，传入了部分参数

``` shell
function m()
(
    _trigger_build "all-modules" "$@"
)
```



从下方分析可以得知，核心逻辑在_wrap_build中

```shell
function _trigger_build()
(
	# 将第一个参数赋值给局部变量 bc（-r表示不可修改）。
	# shift：移除已处理的第一个参数，后续 $@将指向剩余参数
    local -r bc="$1"; shift
    # 获取工程路径
    local T=$(gettop)
    if [ -n "$T" ]; then
      # 能定位到工程路径, 执行脚本
      _wrap_build "$T/build/soong/soong_ui.bash" --build-mode --${bc} --dir="$(pwd)" "$@"
    else
      # 定位不到工程路径，直接返回
      >&2 echo "Couldn't locate the top of the tree. Try setting TOP."
      return 1
    fi
    local ret=$?
    # 所ret为0, ANDROID_QUIET_BUILD为空，ANDROID_BUILD_BANNER非空，则打印输出
    if [[ ret -eq 0 &&  -z "${ANDROID_QUIET_BUILD:-}" && -n "${ANDROID_BUILD_BANNER}" ]]; then
      echo "${ANDROID_BUILD_BANNER}"
    fi
    return $ret
)

```



可以发现_wrap_build只是为构建输出了部分的日志，打印了开始时间、结束时间、构建时长等信息。而触发构建的核心就一行"$@"

而$@是_trigger_build内传入的，具体我们下面进行分析

```shell
function _wrap_build()
{
    # 静默执行
    if [[ "${ANDROID_QUIET_BUILD:-}" == true ]]; then
      "$@"
      return $?
    fi
    # 记录开始时间
    local start_time=$(date +"%s")
    "$@"
    # 获取执行结果
    local ret=$?
    # 获取结束时间
    local end_time=$(date +"%s")
    # .... 懒得分析了，不过可以知道的是是构建过程中的信息。
    local tdiff=$(($end_time-$start_time))
    local hours=$(($tdiff / 3600 ))
    local mins=$((($tdiff % 3600) / 60))
    local secs=$(($tdiff % 60))
    local ncolors=$(tput colors 2>/dev/null)
    if [ -n "$ncolors" ] && [ $ncolors -ge 8 ]; then
        color_failed=$'\E'"[0;31m"
        color_success=$'\E'"[0;32m"
        color_warning=$'\E'"[0;33m"
        color_reset=$'\E'"[00m"
    else
        color_failed=""
        color_success=""
        color_reset=""
    fi

    echo
    if [ $ret -eq 0 ] ; then
        echo -n "${color_success}#### build completed successfully "
    else
        echo -n "${color_failed}#### failed to build some targets "
    fi
    if [ $hours -gt 0 ] ; then
        printf "(%02g:%02g:%02g (hh:mm:ss))" $hours $mins $secs
    elif [ $mins -gt 0 ] ; then
        printf "(%02g:%02g (mm:ss))" $mins $secs
    elif [ $secs -gt 0 ] ; then
        printf "(%s seconds)" $secs
    fi
    echo " ####${color_reset}"
    echo
    return $ret
}
```



我们回过头看一下m，_trigger_build，_wrap_build很容易就知道构建的参数是什么

$projectDir/build/soong/soong_ui.bash --build-mode --all-modules --dir="$(pwd)"



样例：/Users/rose/aosp/android-14.0.0_r73/build/soong/soong_ui.bash --build-mode --all-modules --dir=/Users/rose/aosp/android-14.0.0_r73

```shell
function m()
(
    _trigger_build "all-modules" "$@"
)

function _trigger_build()
(
	# ......
    wrap_build "$T/build/soong/soong_ui.bash" --build-mode --${bc} --dir="$(pwd)" "$@"
      
)

function _wrap_build()
{

	"$@"

}

```



# soong_ui.bash



``` shell
# To track how long we took to startup.
# 记录时间
case $(uname -s) in
  Darwin)
    export TRACE_BEGIN_SOONG=`$T/prebuilts/build-tools/path/darwin-x86/date +%s%3N`
    ;;
  *)
    export TRACE_BEGIN_SOONG=$(date +%s%N)
    ;;
esac

# bash source为soong_ui.bash的绝对地址
# 这行代码就是进入到soong_ui.bash的同层级路径，并执行../make/shell_utils.sh脚本
# shell_utils在分析envsetup.sh已经分析过了，不额外进行阐述了。
source $(cd $(dirname $BASH_SOURCE) &> /dev/null && pwd)/../make/shell_utils.sh
# 记录工程路径
require_top

# Save the current PWD for use in soong_ui
export ORIGINAL_PWD=${PWD}
export TOP=$(gettop)
source ${TOP}/build/soong/scripts/microfactory.bash

soong_build_go soong_ui android/soong/cmd/soong_ui
soong_build_go mk2rbc android/soong/mk2rbc/mk2rbc
soong_build_go rbcrun rbcrun/rbcrun

cd ${TOP}
exec "$(getoutdir)/soong_ui" "$@"
```



## microfactory.bash



```shell
# 寻找go path路径
case $(uname) in
    Linux)
        export GOROOT="${TOP}/prebuilts/go/linux-x86/"
        ;;
    Darwin)
        export GOROOT="${TOP}/prebuilts/go/darwin-x86/"
        ;;
    *) echo "unknown OS:" $(uname) >&2 && exit 1;;
esac

# Find the output directory
# 获取soong_ui路径
# 实例：/Users/rose/aosp/android-14.0.0_r73/out
function getoutdir
{
    local out_dir="${OUT_DIR-}"
    if [ -z "${out_dir}" ]; then
        if [ "${OUT_DIR_COMMON_BASE-}" ]; then
            out_dir="${OUT_DIR_COMMON_BASE}/$(basename ${TOP})"
        else
            out_dir="out"
        fi
    fi
    if [[ "${out_dir}" != /* ]]; then
        out_dir="${TOP}/${out_dir}"
    fi
    echo "${out_dir}"
}

# Bootstrap microfactory from source if necessary and use it to build the
# requested binary.
#
# Arguments:
#  $1: name of the requested binary
#  $2: package name
function soong_build_go
{
    BUILDDIR=$(getoutdir) \
      SRCDIR=${TOP} \
      BLUEPRINTDIR=${TOP}/build/blueprint \
      EXTRA_ARGS="-pkg-path android/soong=${TOP}/build/soong -pkg-path prebuilts/bazel/common/proto=${TOP}/prebuilts/bazel/common/proto -pkg-path rbcrun=${TOP}/build/make/tools/rbcrun -pkg-path google.golang.org/protobuf=${TOP}/external/golang-protobuf -pkg-path go.starlark.net=${TOP}/external/starlark-go" \
      build_go $@
}

source ${TOP}/build/blueprint/microfactory/microfactory.bash

```



## 另一个microfactory.bash



``` shell
function build_go
{
    # Increment when microfactory changes enough that it cannot rebuild itself.
    # For example, if we use a new command line argument that doesn't work on older versions.
    local mf_version=3

    local mf_src="${BLUEPRINTDIR}/microfactory"
    local mf_bin="${BUILDDIR}/microfactory_$(uname)"
    local mf_version_file="${BUILDDIR}/.microfactory_$(uname)_version"
    local built_bin="${BUILDDIR}/$1"
    local from_src=1

    if [ -f "${mf_bin}" ] && [ -f "${mf_version_file}" ]; then
        if [ "${mf_version}" -eq "$(cat "${mf_version_file}")" ]; then
            from_src=0
        fi
    fi

    local mf_cmd
    if [ $from_src -eq 1 ]; then
        # `go run` requires a single main package, so create one
        local gen_src_dir="${BUILDDIR}/.microfactory_$(uname)_intermediates/src"
        mkdir -p "${gen_src_dir}"
        sed "s/^package microfactory/package main/" "${mf_src}/microfactory.go" >"${gen_src_dir}/microfactory.go"
        printf "\n//for use with go run\nfunc main() { Main() }\n" >>"${gen_src_dir}/microfactory.go"

        mf_cmd="${GOROOT}/bin/go run ${gen_src_dir}/microfactory.go"
    else
        mf_cmd="${mf_bin}"
    fi

    rm -f "${BUILDDIR}/.$1.trace"
    # GOROOT must be absolute because `go run` changes the local directory
    GOROOT=$(cd $GOROOT; pwd) ${mf_cmd} -b "${mf_bin}" \
            -pkg-path "github.com/google/blueprint=${BLUEPRINTDIR}" \
            -trimpath "${SRCDIR}" \
            ${EXTRA_ARGS} \
            -o "${built_bin}" $2

    if [ $? -eq 0 ] && [ $from_src -eq 1 ]; then
        echo "${mf_version}" >"${mf_version_file}"
    fi
}
```



## bootstrap



### soog_ui



```shell
/Users/rose/aosp/android-14.0.0_r73/out/microfactory_Linux 
-b /Users/rose/aosp/android-14.0.0_r73/out/microfactory_Linux           
-pkg-path github.com/google/blueprint=/Users/rose/aosp/android-14.0.0_r73/build/blueprint             
-trimpath /Users/rose/aosp/android-14.0.0_r73             
-pkg-path android/soong=/Users/rose/aosp/android-14.0.0_r73/build/soong 
-pkg-path prebuilts/bazel/common/proto=/Users/rose/aosp/android-14.0.0_r73/prebuilts/bazel/common/proto 
-pkg-path rbcrun=/Users/rose/aosp/android-14.0.0_r73/build/make/tools/rbcrun 
-pkg-path google.golang.org/protobuf=/Users/rose/aosp/android-14.0.0_r73/external/golang-protobuf 
-pkg-path go.starlark.net=/Users/rose/aosp/android-14.0.0_r73/external/starlark-go             
-o /Users/rose/aosp/android-14.0.0_r73/out/soong_ui android/soong/cmd/soong_ui
```







### mk2rbc



```shell
/Users/rose/aosp/android-14.0.0_r73/out/microfactory_Linux 
-b /Users/rose/aosp/android-14.0.0_r73/out/microfactory_Linux             
-pkg-path github.com/google/blueprint=/Users/rose/aosp/android-14.0.0_r73/build/blueprint             
-trimpath /Users/rose/aosp/android-14.0.0_r73             
-pkg-path android/soong=/Users/rose/aosp/android-14.0.0_r73/build/soong 
-pkg-path prebuilts/bazel/common/proto=/Users/rose/aosp/android-14.0.0_r73/prebuilts/bazel/common/proto 
-pkg-path rbcrun=/Users/rose/aosp/android-14.0.0_r73/build/make/tools/rbcrun 
-pkg-path google.golang.org/protobuf=/Users/rose/aosp/android-14.0.0_r73/external/golang-protobuf 
-pkg-path go.starlark.net=/Users/rose/aosp/android-14.0.0_r73/external/starlark-go             
-o /Users/rose/aosp/android-14.0.0_r73/out/mk2rbc android/soong/mk2rbc/mk2rbc
```





### rbcrun



```shell
/Users/rose/aosp/android-14.0.0_r73/out/microfactory_Linux 
-b /Users/rose/aosp/android-14.0.0_r73/out/microfactory_Linux             
-pkg-path github.com/google/blueprint=/Users/rose/aosp/android-14.0.0_r73/build/blueprint             
-trimpath /Users/rose/aosp/android-14.0.0_r73             
-pkg-path android/soong=/Users/rose/aosp/android-14.0.0_r73/build/soong 
-pkg-path prebuilts/bazel/common/proto=/Users/rose/aosp/android-14.0.0_r73/prebuilts/bazel/common/proto 
-pkg-path rbcrun=/Users/rose/aosp/android-14.0.0_r73/build/make/tools/rbcrun 
-pkg-path google.golang.org/protobuf=/Users/rose/aosp/android-14.0.0_r73/external/golang-protobuf 
-pkg-path go.starlark.net=/Users/rose/aosp/android-14.0.0_r73/external/starlark-go             
-o /Users/rose/aosp/android-14.0.0_r73/out/rbcrun rbcrun/rbcrun
```



# soong_ui



shell脚本

```shell
/Users/rose/aosp/android-14.0.0_r73/out/soong_ui 
--build-mode 
--all-modules 
--dir=/Users/rose/aosp/android-14.0.0_r73
```





## init



所有package下的init的方法都会进行component的注册

以binary.go为例



```go
func init() {
	RegisterBinaryBuildComponents(android.InitRegistrationContext)
}

func RegisterBinaryBuildComponents(ctx android.RegistrationContext) {
	ctx.RegisterModuleType("cc_binary", BinaryFactory)
	ctx.RegisterModuleType("cc_binary_host", BinaryHostFactory)
}
```



### RegisterModuleType



// register.go

```go
func (ctx *initRegistrationContext) RegisterModuleType(name string, factory ModuleFactory) {
    // 确保之前没有注册过
	if _, present := ctx.moduleTypes[name]; present {
		panic(fmt.Sprintf("module type %q is already registered", name))
	}
    // 保存name & factory
	ctx.moduleTypes[name] = factory
    
    
    // 保存factory，moduleTypesForDocs，moduleTypeByFactory，数据结构如下
    
    // moduleType是包含name & factory的结构体
    // var moduleTypes []moduleType
    
    // key为name, value为reflect.ValueOf(factory)
	// var moduleTypesForDocs = map[string]reflect.Value{}
    
    // key为reflect.ValueOf(factory), value为name
	// var moduleTypeByFactory = map[reflect.Value]string{}
	RegisterModuleType(name, factory)
	
    // 好像是一个重复调用，RegisterModuleType已经调用过了
    RegisterModuleTypeForDocs(name, reflect.ValueOf(factory))
}
```



## main



```go
func main() {
	shared.ReexecWithDelveMaybe(os.Getenv("SOONG_UI_DELVE"), shared.ResolveDelveBinary())

	buildStarted := time.Now()

	c, args, err := getCommand(os.Args)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error parsing `soong` args: %s.\n", err)
		os.Exit(1)
	}

	// Create a terminal output that mimics Ninja's.
	output := terminal.NewStatusOutput(c.stdio().Stdout(), os.Getenv("NINJA_STATUS"), c.simpleOutput,
		build.OsEnvironment().IsEnvTrue("ANDROID_QUIET_BUILD"),
		build.OsEnvironment().IsEnvTrue("SOONG_UI_ANSI_OUTPUT"))

	// Create and start a new metric record.
	met := metrics.New()
	met.SetBuildDateTime(buildStarted)
	met.SetBuildCommand(os.Args)

	// Attach a new logger instance to the terminal output.
	log := logger.NewWithMetrics(output, met)
	defer log.Cleanup()

	// Create a context to simplify the program termination process.
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// Create a new trace file writer, making it log events to the log instance.
	trace := tracer.New(log)
	defer trace.Close()

	// Create a new Status instance, which manages action counts and event output channels.
	stat := &status.Status{}

	// Hook up the terminal output and tracer to Status.
	stat.AddOutput(output)
	stat.AddOutput(trace.StatusTracer())

	// Set up a cleanup procedure in case the normal termination process doesn't work.
	signal.SetupSignals(log, cancel, func() {
		trace.Close()
		log.Cleanup()
		stat.Finish()
	})
	criticalPath := status.NewCriticalPath()
	buildCtx := build.Context{ContextImpl: &build.ContextImpl{
		Context:      ctx,
		Logger:       log,
		Metrics:      met,
		Tracer:       trace,
		Writer:       output,
		Status:       stat,
		CriticalPath: criticalPath,
	}}

	freshConfig := func() build.Config {
		config := c.config(buildCtx, args...)
		config.SetLogsPrefix(c.logsPrefix)
		return config
	}
	config := freshConfig()
	logsDir := config.LogsDir()
	buildStarted = config.BuildStartedTimeOrDefault(buildStarted)

	buildErrorFile := filepath.Join(logsDir, c.logsPrefix+"build_error")
	soongMetricsFile := filepath.Join(logsDir, c.logsPrefix+"soong_metrics")
	rbeMetricsFile := filepath.Join(logsDir, c.logsPrefix+"rbe_metrics.pb")
	soongBuildMetricsFile := filepath.Join(logsDir, c.logsPrefix+"soong_build_metrics.pb")

	metricsFiles := []string{
		buildErrorFile,        // build error strings
		rbeMetricsFile,        // high level metrics related to remote build execution.
		soongMetricsFile,      // high level metrics related to this build system.
		soongBuildMetricsFile, // high level metrics related to soong build
	}

	os.MkdirAll(logsDir, 0777)

	log.SetOutput(filepath.Join(logsDir, c.logsPrefix+"soong.log"))

	trace.SetOutput(filepath.Join(logsDir, c.logsPrefix+"build.trace"))

	log.Verbose("Command Line: ")
	for i, arg := range os.Args {
		log.Verbosef("  [%d] %s", i, arg)
	}

	// We need to call preProductConfigSetup before we can do product config, which is how we get
	// PRODUCT_CONFIG_RELEASE_MAPS set for the final product config for the build.
	// When product config uses a declarative language, we won't need to rerun product config.
	preProductConfigSetup(buildCtx, config)
	if build.SetProductReleaseConfigMaps(buildCtx, config) {
		log.Verbose("Product release config maps found\n")
		config = freshConfig()
	}

	defer func() {
		stat.Finish()
		criticalPath.WriteToMetrics(met)
		met.Dump(soongMetricsFile)
		if !config.SkipMetricsUpload() {
			build.UploadMetrics(buildCtx, config, c.simpleOutput, buildStarted, metricsFiles...)
		}
	}()
	c.run(buildCtx, config, args)

}
```





### build.Build

```go
func Build(ctx Context, config Config) {
	ctx.Verboseln("Starting build with args:", config.Arguments())
	ctx.Verboseln("Environment:", config.Environment().Environ())

	ctx.BeginTrace(metrics.Total, "total")
	defer ctx.EndTrace()

	if inList("help", config.Arguments()) {
		help(ctx, config)
		return
	}

	// Make sure that no other Soong process is running with the same output directory
	buildLock := BecomeSingletonOrFail(ctx, config)
	defer buildLock.Unlock()

	logArgsOtherThan := func(specialTargets ...string) {
		var ignored []string
		for _, a := range config.Arguments() {
			if !inList(a, specialTargets) {
				ignored = append(ignored, a)
			}
		}
		if len(ignored) > 0 {
			ctx.Printf("ignoring arguments %q", ignored)
		}
	}

	if inList("clean", config.Arguments()) || inList("clobber", config.Arguments()) {
		logArgsOtherThan("clean", "clobber")
		clean(ctx, config)
		return
	}

	defer waitForDist(ctx)

	// checkProblematicFiles aborts the build if Android.mk or CleanSpec.mk are found at the root of the tree.
    // #1 前置准备check，如果根路径下有Android.mk or CleanSpec.mk 则直接报错退出。
	checkProblematicFiles(ctx)
	// ＃2 RAM 大小check，至少需要16GB
	checkRAM(ctx, config)
	//  #3 确保一些常见的文件存在
	SetupOutDir(ctx, config)

	// checkCaseSensitivity issues a warning if a case-insensitive file system is being used.
    //  #4 确保当前文件系统是大小写敏感的。 
	checkCaseSensitivity(ctx, config)
	//  #5 通过symbolic link将$PATH下的二进制文件链接到out/.path路径下
	SetupPath(ctx, config)
	//  #6 计算后续的执行动作，将后续行为信息保存到what中
	what := evaluateWhatToRun(config, ctx.Verboseln)
	//  #7 
	if config.StartGoma() {
		startGoma(ctx, config)
	}

	rbeCh := make(chan bool)
	var rbePanic any
	if config.StartRBE() {
		cleanupRBELogsDir(ctx, config)
		checkRBERequirements(ctx, config)
		go func() {
			defer func() {
				rbePanic = recover()
				close(rbeCh)
			}()
			startRBE(ctx, config)
		}()
		defer DumpRBEMetrics(ctx, config, filepath.Join(config.LogsDir(), "rbe_metrics.pb"))
	} else {
		close(rbeCh)
	}

	if what&RunProductConfig != 0 {
		runMakeProductConfig(ctx, config)
	}

	// Everything below here depends on product config.

	if inList("installclean", config.Arguments()) ||
		inList("install-clean", config.Arguments()) {
		logArgsOtherThan("installclean", "install-clean")
		installClean(ctx, config)
		ctx.Println("Deleted images and staging directories.")
		return
	}

	if inList("dataclean", config.Arguments()) ||
		inList("data-clean", config.Arguments()) {
		logArgsOtherThan("dataclean", "data-clean")
		dataClean(ctx, config)
		ctx.Println("Deleted data files.")
		return
	}

	if what&RunSoong != 0 {
		runSoong(ctx, config)
	}

	if what&RunKati != 0 {
		genKatiSuffix(ctx, config)
		runKatiCleanSpec(ctx, config)
		runKatiBuild(ctx, config)
		runKatiPackage(ctx, config)

		ioutil.WriteFile(config.LastKatiSuffixFile(), []byte(config.KatiSuffix()), 0666) // a+rw
	} else if what&RunKatiNinja != 0 {
		// Load last Kati Suffix if it exists
		if katiSuffix, err := ioutil.ReadFile(config.LastKatiSuffixFile()); err == nil {
			ctx.Verboseln("Loaded previous kati config:", string(katiSuffix))
			config.SetKatiSuffix(string(katiSuffix))
		}
	}

	// Write combined ninja file
	createCombinedBuildNinjaFile(ctx, config)

	distGzipFile(ctx, config, config.CombinedNinjaFile())

	if what&RunBuildTests != 0 {
		testForDanglingRules(ctx, config)
	}

	<-rbeCh
	if rbePanic != nil {
		// If there was a ctx.Fatal in startRBE, rethrow it.
		panic(rbePanic)
	}

	if what&RunNinja != 0 {
		if what&RunKati != 0 {
			installCleanIfNecessary(ctx, config)
		}
		runNinjaForBuild(ctx, config)
	}

	if what&RunDistActions != 0 {
		runDistActions(ctx, config)
	}
}
```







### runMakeProductConfig 





### runSong



### runKatiBuild





### runKatiPackage





### createCombinedBuildNinjaFile





### runNinja







# refs



在此感谢所有引用文章

https://notion.iliuqi.com/article/make_of_android_compilation_principle







