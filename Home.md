# Cheat Sheet

This Cheat Sheet contains all commands and code examples used in the **Arm Research Starter Kit (RSK) on System Modeling using gem5**

# 1. Get Started

Follow these instructions to download all the required materials, build a gem5 binary and run simple examples in both System Call Emulation (SE) and Full System (FS) modes.   

## Get Arm Research Starter Kit

### Clone Repositories

##### Get and run the `clone.sh` script to clone both `gem5` and `arm-gem5-rsk` repositories
```bash
$ wget https://raw.githubusercontent.com/arm-university/arm-gem5-rsk/master/clone.sh
$ bash clone.sh
```

### Build gem5

##### Build gem5 binaries for Arm
```bash
$ cd gem5
$ scons build/ARM/gem5.opt -j4 # parallel build
```

## Simulate Arm in gem5

### SE Simulation

##### Run simple SE:
```bash
$ ./build/ARM/gem5.opt configs/example/arm/starter_se.py --cpu="minor" "tests/test-progs/hello/bin/arm/linux/hello" # Hello world!
```

##### Run multicore SE:
>###### **Note**: each core runs its own program
```bash
$ ./build/ARM/gem5.opt configs/example/arm/starter_se.py --cpu="minor" --num-cores=2 "tests/test-progs/hello/bin/arm/linux/hello" "tests/test-progs/hello/bin/arm/linux/hello"
```

### FS Simulation

##### Get an Arm full system disk image from the [gem5 downloads page](https://www.gem5.org/documentation/general_docs/fullsystem/guest_binaries), and set `$M5_PATH`:
>###### **Note**: the download URLs and filenames below are examples, and may change as new binaries are released by the gem5 project. Please also use a suitable `$M5_PATH` of your choice.
```bash
$ export M5_PATH="${PWD}/../m5_binaries"
$ mkdir -p "${M5_PATH}"
$ wget -P "${M5_PATH}" http://dist.gem5.org/dist/current/arm/aarch-system-201901106.tar.bz2
$ tar -xjf "${M5_PATH}/aarch-system-201901106.tar.bz2" -C "${M5_PATH}"
$ wget -P "${M5_PATH}/disks" http://dist.gem5.org/dist/current/arm/disks/linaro-minimal-aarch64.img.bz2
$ bunzip2 "${M5_PATH}/disks/linaro-minimal-aarch64.img.bz2"
$ echo "export M5_PATH=${M5_PATH}" >> ~/.bashrc
```

##### Run simple FS:
```bash
$ ./build/ARM/gem5.opt configs/example/arm/starter_fs.py --cpu="minor"
```

##### Run with disk image:
```bash
$ ./build/ARM/gem5.opt configs/example/arm/starter_fs.py --cpu="minor" --num-cores=1 --disk-image=$M5_PATH/disks/linaro-minimal-aarch64.img
```

##### Create a checkpoint after boot:
```bash
$ telnet localhost 3456
$ m5 checkpoint
```

>###### **Note**: `3456` is the default port number used by the gem5 simulator. If this does not work, the correct port number can be found in the gem5 output:

```
system.terminal: Listening for connections on port 3456
```

A more modern alternative is to use the [m5term](https://www.gem5.org/documentation/general_docs/fullsystem/m5term) utility provided with gem5.

##### Restore from a checkpoint:
>###### **Note**: Arm `starter_fs.py` allows you to specify checkpoints with `--restore=m5out/cpt.TICKNUMBER/`. Make sure the `--num-cores` and `--disk-image` arguments are the same as the original simulation; the `--cpu` type may be changed.
```bash
$ ./build/ARM/gem5.opt configs/example/arm/starter_fs.py --restore=m5out/cpt.TICKNUMBER/ --cpu="minor" --num-cores=1 --disk-image=$M5_PATH/disks/linaro-minimal-aarch64.img
```

# 2. SE Benchmarks

We use the Stanford SingleSource workloads from the LLVM test-suite for benchmarking in the SE mode.

## Get and Compile LLVM test-suite

##### Download the Stanford SingleSource LLVM test-suite:
```bash
$ svn export https://llvm.org/svn/llvm-project/test-suite/tags/RELEASE_390/final/SingleSource/Benchmarks/Stanford/ se-benchmarks
```

##### Install the Arm cross compiler toolchain:
```bash
$ sudo apt-get install gcc-aarch64-linux-gnu
```

##### Replace `se-benchmarks/Makefile` with the code below and then `make`:
>###### **Note**: replace indentation whitespaces with tabs in your `Makefile`
```bash
SRCS = $(wildcard *.c)
PROGS = $(patsubst %.c,%,$(SRCS))
all: $(PROGS)
%: %.c
      aarch64-linux-gnu-gcc --static $< -o $@
clean:
      rm -f $(PROGS)
```

## Run LLVM test-suite 

##### Run LLVM test-suite on HPI:
>###### **Note**: set `<benchmark>` and `/path_to_benchmark`
```bash
$ ./build/ARM/gem5.opt -d se_results/<benchmark> configs/example/arm/starter_se.py --cpu="hpi" /path_to_benchmark
```

>###### For example, to run the `Bubblesort` benchmark, run the following command:
```bash
$ ./build/ARM/gem5.opt -d se_results/Bubblesort configs/example/arm/starter_se.py --cpu="hpi" /path_to/se-benchmarks/Bubblesort
```

>###### **Note**: some of the benchmarks may take a very long time to run.

### Read results of SE runs
To compare the benchmarks, we first need to create a simple configuration file and specify a list of benchmarks to be compared, comparison parameters and an output file. Then, we just pass this configuration file to the `read_results.sh` bash script.

##### Step1: create a `.ini` under the `se_results` directory (where the results of SE runs are stored), e.g. `exe_time.ini` looks like this:

 ```bash
[Benchmarks]
Bubblesort
IntMM
Oscar

[Parameters]
sim_seconds

[Output]
res_exe_time.txt
```

##### Step2: run the `read_results.sh` bash script from the `se_results` directory and pass the sample `exe_time.ini` file as a parameter:
```bash
$ cd se_results # where the results of SE runs are stored
$ bash ../../arm-gem5-rsk/read_results.sh exe_time.ini
$ cat res_exe_time.txt
```

# 3. FS Benchmarks

We use The PARSEC Benchmark Suite for benchmarking in the FS mode. 

## Get and Compile PARSEC 

##### Download PARSEC 3.0:
```bash
$ wget http://parsec.cs.princeton.edu/download/3.0/parsec-3.0.tar.gz
$ tar -xvzf parsec-3.0.tar.gz
```

We describe two ways to compile PARSEC benchmarks for Arm: 
* Cross-compiling on an x86 machine
* Compiling on QEMU

The common steps apply to both approaches.

### Common Steps

##### Step1: from the `parsec-3.0` directory, apply `static-patch.diff`: 
```bash
$ patch -p1 < ../arm-gem5-rsk/parsec_patches/static-patch.diff
```

##### Step2: replace `config.guess` and `config.sub`:
>###### **Note**: set path to `/parsec_dir/` and `/absolute_path_to_tmp/`
```bash
$ mkdir tmp; cd tmp # make a tmp dir outside the parsec dir
$ wget -O config.guess 'http://git.savannah.gnu.org/gitweb/?p=config.git;a=blob_plain;f=config.guess;hb=HEAD'
$ wget -O config.sub 'http://git.savannah.gnu.org/gitweb/?p=config.git;a=blob_plain;f=config.sub;hb=HEAD'
$ cd /parsec_dir/ # cd to the parsec dir
$ find . -name "config.guess" -type f -print -execdir cp {} config.guess_old \;
$ find . -name "config.guess" -type f -print -execdir cp /absolute_path_to_tmp/config.guess {} \;
$ find . -name "config.sub" -type f -print -execdir cp {} config.sub_old \;
$ find . -name "config.sub" -type f -print -execdir cp /absolute_path_to_tmp/config.sub {} \;
``` 

### Cross-Compile PARSEC on x86

##### Get the `aarch64-linux-gnu` toolchain:
```bash
$ wget https://releases.linaro.org/components/toolchain/binaries/latest-5/aarch64-linux-gnu/gcc-linaro-5.5.0-2017.10-x86_64_aarch64-linux-gnu.tar.xz
$ tar xvfJ gcc-linaro-5.5.0-2017.10-x86_64_aarch64-linux-gnu.tar.xz
```

##### From the `parsec-3.0` directory, apply `xcompile-patch.diff`:
>###### **Note**: before applying the patch, change the `CC_HOME` and the `BINUTIL_HOME` in the `xcompile-patch.diff` to point to the downloaded `<gcc-linaro directory>` and  `<gcc-linaro directory>/aarch64-linux-gnu` directories
```bash
$ patch -p1 < ../arm-gem5-rsk/parsec_patches/xcompile-patch.diff
```

##### Cross-compile PARSEC:
>###### **Note**: set `<pkgname>`
```bash
$ export PARSECPLAT="aarch64-linux" # set the platform 
$ source ./env.sh
$ parsecmgmt -a build -c gcc-hooks -p <pkgname>
```

### Compile PARSEC on QEMU

##### From the `parsec-3.0` directory, apply `qemu-patch.diff`:
```bash
$ patch -p1 < ../arm-gem5-rsk/parsec_patches/qemu-patch.diff
```

##### Resolve dependencies
```bash
$ sudo apt-get install libglib2.0 libpixman-1-dev libfdt-dev libcap-dev libattr1-dev libcap-ng-dev
```

##### Clone and make QEMU:
```bash
$ git clone git://git.qemu.org/qemu.git qemu
$ cd qemu
$ ./configure --target-list=aarch64-softmmu --enable-virtfs
$ make
```

##### Get the Arm AArch64 kernel and disk image for QEMU:
```bash
$ wget http://releases.linaro.org/archive/15.06/openembedded/aarch64/Image 
$ wget http://releases.linaro.org/archive/15.06/openembedded/aarch64/vexpress64-openembedded_lamp-armv8-gcc-4.9_20150620-722.img.gz 
$ gzip -dc vexpress64-openembedded_lamp-armv8-gcc-4.9_20150620-722.img.gz > vexpress_arm64.img
```

##### Start QEMU:
>###### **Note**: set path to `/shared_directory/`
```bash
$ ./qemu/aarch64-softmmu/qemu-system-aarch64 -m 1024 -cpu cortex-a53 -nographic -machine virt -kernel Image -append 'root=/dev/vda2 rw rootwait mem=1024M console=ttyAMA0,38400n8' -drive if=none,id=image,file=vexpress_arm64.img -netdev user,id=user0 -device virtio-net-device,netdev=user0 -device virtio-blk-device,drive=image -fsdev local,id=r,path=/shared_directory/,security_model=none -device virtio-9p-device,fsdev=r,mount_tag=r
```
##### Mount `/shared_directory/` on QEMU:
```bash
$ mount -t 9p -o trans=virtio r /mnt
```

##### Compile PARSEC on QEMU:
>###### **Note**: set `<pkgname>`
```bash
$ cd /mnt # cd to the mounted PARSEC directory
$ source ./env.sh
$ parsecmgmt -a build -c gcc-hooks -p <pkgname>
$ poweroff # quit QEMU
```

## Run PARSEC

To run PARSEC benchmarks on Arm, we need to enlarge the disk image and copy the directory of compiled benchmarks `parsec-3.0` to it. Then, we need to generate benchmark runscripts and pass them via the `--script` option to the simulation script `starter_fs.py`.

### Copy PARSEC to Disk Image

##### Expand and resize the disk image:
```bash
$ cp linaro-minimal-aarch64.img expanded-linaro-minimal-aarch64.img
$ dd if=/dev/zero bs=1G count=20 >> ./expanded-linaro-minimal-aarch64.img # add 20G zeros
$ sudo parted expanded-linaro-minimal-aarch64.img resizepart 1 100% # grow partition 1
```

##### Mount the expanded disk image, resize the filesystem, and copy PARSEC to it:

###### In order to mount the disk image, we must calculate the offset of the first partition.
###### Run `fdisk` to find the start offset (in sectors) and the sector size (in bytes) of the partition.
```bash
$ fdisk -l expanded-linaro-minimal-aarch64.img
Disk expanded-linaro-minimal-aarch64.img: 21 GiB, 22548316160 bytes, 44039680 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x3e7a980d

Device                               Boot Start      End  Sectors Size Id Type
expanded-linaro-minimal-aarch64.img1         63 44039679 44039617  21G 83 Linux
```
###### In this example  the start offset (in sectors) is 63 and the sector size (in bytes) is 512, giving a start offset of 63x512=*32256* bytes.

###### Now mount the disk image, resize the filesystem, and copy PARSEC.
>###### **Note**: set `/path_to_compiled_parsec-3.0_dir/`, and change `32256` to the offset in bytes calculted above
```bash
$ mkdir disk_mnt
$ sudo mount -o loop,offset=32256 expanded-linaro-minimal-aarch64.img disk_mnt
$ df # find /dev/loopX for disk_mnt
$ sudo resize2fs /dev/loopX # resize filesystem
$ df # check that the Available space for disk_mnt is increased
$ sudo cp -r /path_to_compiled_parsec-3.0_dir/ disk_mnt/home/root # copy the compiled parsec-3.0 to the image
$ ls disk_mnt/home/root # check the parsec-3.0 contents
$ sudo umount disk_mnt
```

### Run PARSEC on HPI
We only show how to run PARSEC on the HPI model. Reading results from `stats.txt` files can be done by using the `read_results.sh` script, as shown for the SE mode.

##### Generate benchmark runscripts
```bash
$ cd arm-gem5-rsk/parsec_rcs
$ bash gen_rcs.sh -p <pkgname> -i <simsmall/simmedium/simlarge> -n <nth>
```

##### Run `starter_fs` using the expanded image and benchmark runscript
>###### **Note**: set both `<benchmark>` instances
```bash
$ ./build/ARM/gem5.opt -d fs_results/<benchmark> configs/example/arm/starter_fs.py --cpu="hpi" --num-cores=1 --disk-image=$M5_PATH/disks/expanded-linaro-minimal-aarch64.img --script=../arm-gem5-rsk/parsec_rcs/<benchmark>.rcS
# Example: canneal on 2 cores
$ ./build/ARM/gem5.opt -d fs_results/canneal_simsmall_2 configs/example/arm/starter_fs.py --cpu="hpi" --num-cores=2 --disk-image=$M5_PATH/disks/expanded-linaro-minimal-aarch64.img --script=../arm-gem5-rsk/parsec_rcs/canneal_simsmall_2.rcS
```
