第 3 章 临时系统
####################

## 3.1 简介

如果之前的章节只能算是介绍和准备工作的话，从本章节开始，我们就要逐渐步入构建 LFS 的过程中了。

本章可以看作是构建 LFS 系统过程中的一个过渡过程。在前章我们已经为构建做好了准备，但是我们还不能直接在当前的环境中构建 LFS，我们需要先构建一个临时系统，然后在这个临时系统中构建 LFS。之所以有这一步显然不是为了提升构建过程的复杂度，而是为了让整个构建的过程更可控，更安全，更方便。

我们在准备阶段已经见识到了，想要达到构建 LFS 的要求，说难不难，说简单也不算简单，当然这个难易度主要取决于你的宿主系统是否让你省心。在本章你也会陆续见识到宿主环境的不同给你带来不同体验，有些发行版就是会有这样那样的问题。当然其中的大部分是可忽略的，不过如果没有这么一个临时系统作为过渡，就直接进行 LFS 的构建工作，这无疑是给构建者徒添痛苦。而对于 LFS 手册的维护和开发也将是严峻的考验。

纵览第 3 章，其主要目的就是构建一个临时的系统。然后，我们再通过 chroot，在该环境中执行其余章节的命令。以保证目标 LFS 系统能够洁净且无故障地生成。该构建过程的设计就是为了让构建者承担最少的风险，同时还能有最好的指导价值。

构建这个最小系统有两个步骤。第一步是构建一个与宿主系统无关的工具链，该工具链包含编译器、汇编器、链接器、库，还有若干实用工具。而第二步则是使用工具链去构建那些必要的工具。

本章中编译得到的所有文件都将存放在 $LFS/tools 目录中，以确保在下一章中安装的文件和宿主系统生成的目录相互分离。由于本章生成的软件包都只是临时性的，因此，更倾向于将它们和后续章节即将构成的 LFS 系统区分开。

## 3.2 通用编译指南

在开始构建前，稍微讲一下安装每个软件包的顺序。其实，放平时安装这些软件包的时候并没有这么多的规矩，但在本章及后续章节中要一次性构建很多软件包，而且有些软件包需要被编译多次，所以请大家务必按下述顺序进行构建操作。

> 1. 使用 tar 指令，解压软件包。（在本章节中，确保你运行解压指令的时候，使用的是 lfs 用户。）
> 2. 进入解压得到的目录中。
> 3. 根据操作指导和说明进行编译。
> 4. 返回 /mnt/lfs/sources/ 目录。
> 5. 删除解压得到的目录。

还有几点需要注意的，在开始前再自检一下：

> 1. 当前宿主环境已满足第 2 章宿主系统要求。
> 2. 所有软件包的源文件和补丁存放在 chroot  环境可访问的目录。例如， /mnt/lfs/sources/。
> 3. 输入 echo $LFS 检查一下环境变量 LFS。如果你完全按照本书指导操作，正确的输出结果应该是 /mnt/lfs。

## 3.3 构建工具链

### 3.3.1 Binutils 交叉编译

构建用时: 1 SBU
磁盘空间: 580 MB

从本小节开始，我们将采用 SBU 作为时间度量值来统计构建所需耗费的时间。SBU 是 Standard Build Unit 的缩写，即标准构建单元的意思，也是源自 LFS 手册对于构建时间的一种衡量方式。由于构建时间对于每个机器来说并不固定，换言之就是，你的计算机快构建的时间自然就快。所以为了给每位构建提供相对准确的构建时间，SBU 就诞生了。SBU 就是本小节构建 Binutils 的时间。注意是本小节构建 Binutils 的时间，我们在构建过程种会多次编译 Binutils，我们仅用本小节的构建时间作为一个标准构建单元，即 1 SBU。而其他软件包的构建耗时也只需要按照本章节的构建时间推算即可获得。举个例子，如果你在本小节的构建工程中消耗了 1 分钟，那么下个小节的交叉编译的 GCC 将需要 11 SBU，也就是 11 分钟。第 3 章的 SBU 值和磁盘空间均是不包含测试套件的数值，后续章节如包含必要的测试，会另行标注测试需要花费的时间。

下面开始构建工作，其中所有的命令以及参数，我会在初见的时候详细的介绍一番，而在第二次出现的时候就不再作过多解释了。如果你对这方面知识不是特别了解，建议慢慢看下去，如果后面有什么不理解的话就往前面翻翻。如果你觉得这些很简单，你也可以跳过，直接运行指令即可，指令已经写的非常完善了，在途中不中断的情况下，直接运行指令即可完成构建工作。不过有一点需要注意，本书为了兼顾 LFS 的 System V 和 systemd 两个版本，两个版本本身是不能共存的，所以在某些软件包的构建和个别软件包的构建顺序上会存在些许差异，这些地方需要读者自己考虑，自己抉择。

首先，进入 $LFS/sources 目录。

> cd $LFS/sources

cd 命令，也称为 chdir，该命令用于更改各种操作系统中的当前工作目录。$LFS 就是环境变量 LFS，所以整个命令的意思就是进入 /mnt/lfs/sources/ 目录。后续章节的命令默认你在构建完上一个软件包后，已经回到了该目录。所以在构建途中，你由于某种缘故切换到别的目录的话，请在执行下一个软件包构建的解压命令前请保证已重新进入了该目录。

解压并进入软件包：

> tar -xf binutils-2.32.tar.xz
> cd binutils-2.32

tar 是 Unix 和类 Unix 系统上的压缩打包工具，可以将多个文件合并为一个文件，打包后的文件常以 tar 结尾。这里的 -xf 的意思是：-x（或 --extract，--get）意为解开 tar 文件；-f（或 --file）后接文件名以指定要处理的文件名。所以上述的第一条指令意为解压 binutils-2.32.tar.xz。而第二条命令和上一条相同，即进入解压所得 binutils-2.32 目录。软件包和解压所得的目录中的 -2.32 是软件包的当前版本，如果你安装的版本比命令中要更新，请自行修改指令。

根据 Binutils 的文档建议，我们最好创建一个区别于源码的独立目录，然后在创建的独立目录中运行编译指令。

> mkdir -v build
> cd       build

mkdir 用于创建目录，-v 参数会将每一创建的目录显示出来。

本小节开始的时候提到过 SBU，如果你想要测试一下 1 SBU 的具体时间，可以使用 time 命令包裹住编译指令 configure、 make 和 make install 测算时间。

> time { 
> ../configure --prefix=/tools                                              \
> --with-sysroot=$LFS                                                       \
> --with-lib-path=/tools/lib                                                \
> --target=$LFS_TGT                                                         \
> --disable-nls                                                             \
> --disable-werror                                                          \
> && make 
> case $(uname -m) in
>   x86_64) mkdir -v /tools/lib && ln -sv lib /tools/lib64 ;;
> esac                                                                      \
> && make install
> }

如果你运行了上述指令，那么直到 make install 命令为止，你都无需重复运行了。

编译前准备：

> ../configure --prefix=/tools            \
>              --with-sysroot=$LFS        \
>              --with-lib-path=/tools/lib \
>              --target=$LFS_TGT          \
>              --disable-nls              \
>              --disable-werror

这里要尤为注意 configure 前面是 [..] 而不是 [.],[.] 代表的是当前目录而 [..] 代表的是上一级目录。我们现在处于 build 目录，其中并没有任何源码文件，所以要在上一级目录运行 configure。如果你并不是在 binutils-2.32 目录下创建 build 目录的，请自行修改命令。

configure 参数的含义：

*--prefix=/tools* 告诉配置脚本将 Binutils 程序安装到 /tools 文件夹。

*--with-sysroot=$LFS* 用于交叉编译，告诉编译系统在 $LFS 中查找所需的目标系统库。

*--with-lib-path=/tools/lib* 为链接器指定库的路径。

*--target=$LFS_TGT* 因为 LFS_TGT 变量中的机器描述和 config.guess 脚本返回的值略有不同，这个选项会告诉 configure 脚本调整 binutils 的构建系统来构建一个交叉链接器。

*--disable-nls* 禁止国际化（i18n），因为国际化对临时工具来说没有必要。

*--disable-werror* 防止来自宿主编译器的警告事件导致编译停止。

编译软件包：

> make

如果是在 x86_64 上构建，为确保工具链的完整性，需要创建符号链接：

> case $(uname -m) in
>   x86_64) mkdir -v /tools/lib && ln -sv lib /tools/lib64 ;;
> esac

命令中包含对 x86_64 的判断，仅在 x86_64 的情况下执行创建工作。但是实现搞清楚自己的环境也是件蛮重要的事情，可以遵从命令输入 uname -m 查看，后续章节也有类似的指令,如果输出的不是 x86_64 那也就不用浪费时间了。

安装软件包：

> make install

通过本小节的标题可知，本次安装的是交叉编译的 Binutils，之后我们还将交叉编译并安装 GCC。然后是经过净化的 Linux API 头文件和 Glibc。然后我们还要在安装一遍 Binutils 和 GCC。之所以我们要先交叉编译并安装 Binutils 和 GCC 的主要原因是减少临时系统对宿主系统的依赖，同时也消除了宿主系统对于目标系统的潜在污染。

安装完成后，推出并清理软件包：

> cd ../..
> rm -rf binutils-2.32

通过 cd 命令我们会回到 /mnt/lfs/sources/ 目录，本节开头已经提过了，每个软件包构建结束后要回到 /mnt/lfs/sources/ 目录，并删除解压所得的目录。rm 命令用于删除目录 binutils-2.32。-r 参数意为递归删除，就是会事先帮你把目录中的内容删除，然后再删除该目录。-f 参数会无视确认提示，毕竟目录里的文件很多，一个个确认确实十分麻烦。

### 3.3.2 GCC 交叉编译

构建用时: 11 SBU
磁盘空间: 2.9 GB

首先，解压并进入软件包：

> tar -xf gcc-8.2.0.tar.xz
> cd  gcc-8.2.0

然后，解压 GMP、MPFR 和 MPC 这三个软件包，并将解压后的目录重命名。在构建临时系统时，我们需要将它们和 GCC 一起编译。

> tar -xf ../mpfr-4.0.2.tar.xz
> mv -v mpfr-4.0.2 mpfr
> tar -xf ../gmp-6.1.2.tar.xz
> mv -v gmp-6.1.2 gmp
> tar -xf ../mpc-1.1.0.tar.gz
> mv -v mpc-1.1.0 mpc

mv 指令是用于移动文件和目录的，当然如果移动前的目录或文件和移动后的目录或文件位于同一目录，其实际操作就和变名没什么区别了。-v 参数是详述模式，在移动目录或文件时将会列出它们的名字。

运行下述指令以修改 GCC 默认的的位置，从而使用 /tools 目录中的动态链接器。同时会将 /usr/include 从 GCC 的 include 检索路径中移除。

> for file in gcc/config/{linux,i386/linux{,64}}.h
> do
>   cp -uv $file{,.orig}
>   sed -e 's@/lib\(64\)\?\(32\)\?/ld@/tools&@g' \
>       -e 's@/usr@/tools@g' $file.orig > $file
>   echo '
> #undef STANDARD_STARTFILE_PREFIX_1
> #undef STANDARD_STARTFILE_PREFIX_2
> #define STANDARD_STARTFILE_PREFIX_1 "/tools/lib/"
> #define STANDARD_STARTFILE_PREFIX_2 ""' >> $file
>   touch $file.orig
> done

上述命令的内容乍一看确实难以理解，那就让我们逐条解析一下吧！首先时备份 gcc/config/linux.h，gcc/config/i386/linux.h，和 gcc/config/i368/linux64.h，副本文件名称将在原有文件后追加 .orig 后缀。然后，sed 指令的第一个表达式意为在每个 /lib/ld、/lib64/ld 或 /lib32/ld 前追加 /tools，而第二个表达式意为替换 /usr 的硬编码实例。接着，改变 STANDARD_STARTFILE_PREFIX，需要注意 /tools/lib/ 中最后一个 / 是必须的不能省略。最后，使用 touch 更新备份文件的时间戳。cp 命令的 -u 参数会比较复制文件的时间戳，一旦复制的目的文件比源文件更新，就不会进行复制操作。由此我们可以防止命令被无意中运行两次，从而对原始文件造成意外的更改。

在 x86_64 的主机上，为 64 位的库设置默认目录名至 lib：

> case $(uname -m) in
>   x86_64)
>     sed -e '/m64=/s/lib64/lib/' \
>         -i.orig gcc/config/i386/t-linux64 ;;
> esac

GCC 手册也建议在源码目录之外新建一个专门的编译目录：

> mkdir -v build
> cd       build

编译前准备:

> ../configure                                       \
>     --target=$LFS_TGT                              \
>     --prefix=/tools                                \
>     --with-glibc-version=2.11                      \
>     --with-sysroot=$LFS                            \
>     --with-newlib                                  \
>     --without-headers                              \
>     --with-local-prefix=/tools                     \
>     --with-native-system-header-dir=/tools/include \
>     --disable-nls                                  \
>     --disable-shared                               \
>     --disable-multilib                             \
>     --disable-decimal-float                        \
>     --disable-threads                              \
>     --disable-libatomic                            \
>     --disable-libgomp                              \
>     --disable-libmpx                               \
>     --disable-libquadmath                          \
>     --disable-libssp                               \
>     --disable-libvtv                               \
>     --disable-libstdcxx                            \
>     --enable-languages=c,c++

configure 参数的含义：

