---
layout: default
title: A handy Makefile
---
I wrote it under MingW, that's why my action of clean target is "del" rather than "rm". It's simple and clear, so I don't try to explain it.

Have a [GNU Make Manual](http://www.gnu.org/software/make/manual/make.html) by your hand if you are not familiar with the syntax.

    CXXFLAGS=-Wall -g

    LDFLAGS=-lws2_32

    SRCS=$(wildcard *.cpp)

    all: CDRCollector.exe

    CDRCollector.exe: $(SRCS:.cpp=.o)

    $(CXX) $(LDFLAGS) $^ -o $@

    -include $(SRCS:.cpp=.d)

    $(SRCS:.cpp=.d): %.d : %.cpp

    $(CXX) -M $(CXXFLAGS) $ $@

    .PHONY: clean

    clean:

    del *.d *.o

