---
layout: post
title: 【Android】Eclipse JNI开发
date: 2014-04-29 22:01:00
categories: [Android]
tags: []
---
Android Tools->Add Native Support
Run -> External Tools -> javah
Location /usr/bin/javah
Working Directory  ${workspace_loc:/MiioEngine/bin/classes}

Arguments -d  ${workspace_loc:/MiioEngine/jni} com.xiaomi.miioengine.JNIBridge

