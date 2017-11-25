# Cross-compilation instructions - compiling for Windows on Unix

This method should be suitable for any Unix system: Linux, FreeBSD, MacOSX, etc.

While writing this tutorial I used Ubuntu 16.04, so you will have to install the corresponding packages below for your system. Luckily, they aren't a whole lot.

Something to note is you can replace `/home/<your name>` with `~` (tilde) on most Unix systems, to shorten commands.

Another thing - when building for MXE, I recommend using `make -jn` where N is the number of cores on your processor, to enable multi-threading. This makes the process much faster.

### 1) Preparation of the system for cross-compiliation.

Open the terminal and type the following commands:
 
	sudo apt-get install git bison cmake flex g++ gperf ruby scons libghc-zlib-dev  libghc-zlib-bindings-dev

### 2) Installing MXE and dependencies

	cd
	git clone https://github.com/mxe/mxe.git

### 2.1) gcc & co.

Note: for the below commands, use `-jn` to enable multi-threading - replace `n` with the number of cores on your processor.

	cd /home/<your name>/mxe
	make gcc zlib libpng miniupnpc -jn

Verify if gcc was correctly installed:

	cd /home/<your name>/mxe/usr/bin
	./i686-w64-mingw32.static-gcc -v 

The above command should return:

	.....
	Thread model: win32
	gcc version 5.4.0 (GCC)

In the folder `mxe/usr/i686-w64-mingw32.static/lib` there should be the following files:

	libz.a
	libpng.a
	libminiupnpc.a

### 2.2) Openssl

Go back to the mxe folder and:

	make openssl -jn

To verify if Openssl was properly installed, in the folder `mxe/usr/i686-w64-mingw32.static/lib` there should be:

	libssl.a
	libcrypto.a

And in `mxe/usr/i686-w64-mingw32.static/include` there should be the folder `openssl`.

### 2.3) Boost

Go back to the mxe folder and:

	make boost -jn

To verify if Boost was properly installed, in the folder `mxe/usr/i686-w64-mingw32.static/lib` there should be:

	Various "libboost_something-mt.a" files
	Various "libboost_something-mt.da" files

And in `mxe/usr/i686-w64-mingw32.static/include` there should be the folder `boost`.

### 2.4) Berkley DB
Download http://download.oracle.com/berkeley-db/db-6.0.20.tar.gz and extract (`tar xvzf db-6.0.20.tar.gz`) in your home folder.

Edit the file `db-6.0.20/src/dbinc/win_db.h` and change:

	#include <WinIoCtl.h>
to

	#include <winioctl.h>

Save the file and open the terminal:

	cd /home/<your name>/db-6.0.20/build_unix
	export PATH=/home/<your name>/mxe/usr/bin:$PATH
	 ../dist/configure --host=i686-w64-mingw32.static --enable-mingw --enable-cxx --disable-shared --disable-replication
	make -jn

Once that's done, to verify that the bdb installation was successful, in the folder `build_unix` there should be:

	libdb.a
	libdb_cxx.a

### 2.5) qrencode

Download http://fukuchi.org/works/qrencode/qrencode-3.4.4.tar.gz and extract in the home folder.
Go to the terminal and type:

	export PATH=/home/<your user>/mxe/usr/bin:$PATH (you only have to do this if you closed the terminal from the last time, otherwise, ignore this command now and the next times it appears)
	cd qrencode-3.4.4
	./configure --host=i686-w64-mingw32.static --enable-static --disable-shared --without-tools
	make -jn

To verify that qrencode has succesfully been compiled, in the folder `qrencode-3.4.4/.libs` there should be:

	libqrencode.a

### 2.6) Qt 5 and Qt 4

For this specific coin (Ammo Reloaded) only Qt4 is necessary. But many other coins need Qt5.

Go to the MXE folder in the terminal and type:

	#For Qt4:
	make qt -jn

	#For Qt4 and Qt5:
	make qt qtbase qttools qttranslations -jn

This will take a while, so go make some food.

To verify that Qt4 has been succesfully installed, type:

	export PATH=/home/<your user>/mxe/usr/bin:$PATH
	i686-w64-mingw32.static-qmake-qt4 -v

The response should be:

	QMake version 2.01a
	Using Qt version 4.8.7 in /home/<your user>/mxe/usr/i686-w64-mingw32.static/qt/lib

To verify that Qt5 has been succesfully installed, type:

	export PATH=/home/<your user>/mxe/usr/bin:$PATH
	i686-w64-mingw32.static-qmake-qt5 -v

The response should be:

	QMake version 3.0
	Using Qt version 5.5.1 in /home/<your user>/mxe/usr/i686-w64-mingw32.static/qt5/lib

### 3) Compiliation
Before this step please double-check that everything in the previous step has been done correctly, otherwise it is a pain to troubleshoot.

### 3.1) Download source

Open the terminal in your home folder and type:

	git clone https://github.com/ammocore/ammoreloaded 

Or whatever coin you're compiling.

### 3.2) Compiling the coin daemon
Open the file `ammoreloaded/src/makefile.mingw` in a text editor, I recommend a GUI editor for this, not terminal.

Insert the following:

	CC=$(CROSS)gcc
	CXX=$(CROSS)g++

	USE_ASM:=1

Change the `INCLUDEPATHS, LIBPATHS, LIBS` to:


	BOOST_SUFFIX?=-mt
	BOOST_THREAD_LIB_SUFFIX?=_win32-mt

	INCLUDEPATHS= \
	 -I"$(CURDIR)" \
	 -I"/home/<ваше имя>/db-6.0.20/build_unix" \
	 
	LIBPATHS= \
	 -L"$(CURDIR)/leveldb" \
	 -L"/home/<ваше имя>/db-6.0.20/build_unix" \
	
	LIBS= \
	  -l leveldb \
	  -l memenv \
	  -l boost_system$(BOOST_SUFFIX) \
	  -l boost_filesystem$(BOOST_SUFFIX) \
	  -l boost_program_options$(BOOST_SUFFIX) \
	  -l boost_thread$(BOOST_THREAD_LIB_SUFFIX) \
	  -l boost_chrono$(BOOST_SUFFIX) \
	  -l db_cxx \
	  -l ssl \
	  -l crypto \
	  -l z \
	  -l pthread

Make sure there's no whitespace at the end of any of the lines otherwise you will have compile issues. This applies to the next steps as well.

Change the lines:

	g++ -c $(CFLAGS) -o $@ $<
to

	$(CXX) -c $(CFLAGS) -o $@ $<

and

	g++ $(CFLAGS) $(LDFLAGS) -o $@ $(LIBPATHS) $^ $(LIBS)
to

	$(CXX) $(CFLAGS) $(LDFLAGS) -o $@ $(LIBPATHS) $^ $(LIBS)
and

	cd leveldb; make; cd ..
to

	cd leveldb; TARGET_OS=NATIVE_WINDOWS make CROSS=i686-w64-mingw32.static- libleveldb.a libmemenv.a; cd ..
and

	clean:
		-del /Q novacoind.exe
		-del / Q obj \ *
		-del /Q crypto\scrypt\asm\obj\*

to

	clean:
		-rm novacoind.exe
		-rm obj/*
		-rm crypto/scrypt/asm/obj/*

If you want to use LevelDB as the base of the block database, change:

	USE_LEVELDB:=0
to

	USE_LEVELDB:=1

Don't forget to save the file.
Now go to the file `src/leveldb/Makefile`.
Insert after the line:

	CXXFLAGS + = -I. -I./include $ (PLATFORM_CXXFLAGS) $ (OPT)

The following lines:

	CC=$(CROSS)gcc
	CXX=$(CROSS)g++

Save the file.

You should be all set to compile the coin daemon. Open the terminal and type:

	cd /home/<your user>/ammoreloaded/src
	export PATH=/home/<your user>/mxe/usr/bin:$PATH
	make -jn CROSS=i686-w64-mingw32.static- -f makefile.mingw

Now wait for the compile to finish. Once it's done, type:

	strip Ammod.exe

To reduce the file size by ~90%.


### 3.3) Building the graphical wallet (QT)

Go back to `src/leveldb` and type:

	make clean
	TARGET_OS=NATIVE_WINDOWS make CROSS=i686-w64-mingw32.static- libleveldb.a libmemenv.a -jn

Open the file Ammo-qt.pro in a text editor. Once again, I recommend a GUI editor.
Below the lines:

	# Dependency library locations can be customized with:
	#    BOOST_INCLUDE_PATH, BOOST_LIB_PATH, BDB_INCLUDE_PATH,
	#    BDB_LIB_PATH, OPENSSL_INCLUDE_PATH and OPENSSL_LIB_PATH respectively

Replace/add the following:

	BOOST_LIB_SUFFIX=-mt
	BOOST_THREAD_LIB_SUFFIX=_win32-mt
	BDB_INCLUDE_PATH=/home/<your user>/db-6.0.20/build_unix
	BDB_LIB_PATH=/home/<your user>/db-6.0.20/build_unix
	QRENCODE_INCLUDE_PATH=/home/<your user>/qrencode-3.4.4
	QRENCODE_LIB_PATH=/home/<your user>/qrencode-3.4.4/.libs

Also comment out (add a # to the beginning of the line) if it isn't already commented out:

	genleveldb.commands = cd $$PWD/src/leveldb && CC=$$QMAKE_CC CXX=$$QMAKE_CXX TARGET_OS=OS_WINDOWS_CROSSCOMPILE $(MAKE) OPT=\"$$QMAKE_CXXFLAGS $$QMAKE_CXXFLAGS_RELEASE\" libleveldb.a libmemenv.a && $$QMAKE_RANLIB $$PWD/src/leveldb/libleveldb.a && $$QMAKE_RANLIB $$PWD/src/leveldb/libmemenv.a

If the following line:

	CONFIG += static

Does not exist, then add that line to the beginning of the file.

Change:

	win32:QMAKE_LFLAGS........................
to:

	win32:QMAKE_LFLAGS *= -Wl,--large-address-aware -static

(The above is already done on ammo-reloaded but this applies to other coins as well.)

and

	win32:QMAKE_LRELEASE = $$[QT_INSTALL_BINS]\\lrelease.exe
to:

	win32:QMAKE_LRELEASE = $$[QT_INSTALL_BINS]/lrelease

Save the modified Ammo-qt.pro.
Open the terminal and run the following commands:

<b>On all the below commands, you need to open Makefile.Release and add:</b>

	-I/home/<your user>/db-6.0.20/build_unix/

to the end of the `INCPATH` line.

#### For Qt4 + LevelDB (normally this is what you should use):

	export PATH=/home/<your user>/mxe/usr/bin:$PATH
	cd ammo-reloaded
	i686-w64-mingw32.static-qmake-qt4 "USE_IPV6=1" "USE_LEVELDB=1" "USE_ASM=1" Ammo-qt.pro
	#Edit the Makefile as stated above.
	make -jn -f Makefile.Release


#### For Qt4 + BDB:

	export PATH=/home/<your user>/mxe/usr/bin:$PATH
	cd ammo-reloaded
	i686-w64-mingw32.static-qmake-qt4 "USE_IPV6=1" "USE_ASM=1" Ammo-qt.pro
	#Edit the Makefile as stated above.
	make -jn -f Makefile.Release

#### For Qt5 + LevelDB:

	export PATH=/home/<your user>/mxe/usr/bin:$PATH
	cd ammo-reloaded
	i686-w64-mingw32.static-qmake-qt5 "USE_IPV6=1" "USE_LEVELDB=1" "USE_ASM=1" Ammo-qt.pro
	#Edit the Makefile as stated above.
	make -jn -f Makefile.Release

#### For Qt5 + BDB:

	export PATH=/home/<your user>/mxe/usr/bin:$PATH
	cd ammo-reloaded
	i686-w64-mingw32.static-qmake-qt5 "USE_IPV6=1" "USE_ASM=1" Ammo-qt.pro
	#Edit the Makefile as stated above.
	make -jn -f Makefile.Release


After the compile is done, the binary file Ammo-qt.exe will be located in the `release` folder.
Once again, do:

	strip Ammo-qt.exe

to reduce the size by ~90%.


That's it! If you're having compile issues, please google them first and then open an issue here if you can't find the answer. I will help you as soon as possible.
