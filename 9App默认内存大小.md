# App 的默认内存大小

Android 给每个 App 可使用的 Heap 大小设定了一个限定值。这个值是系统设置的 prop 值, 系统编译时内置的, 保存在 `system/build.prop` 中. **一般国内的手机厂商都会做修改, 根据手机配置不同而不同**, 可以通过如下命令查看:

    $ adb shell
    shell@hwH60:/ $ cat /system/build.prop

以手头的 Huawei 荣耀 6 为例, heap size 相关的 prop 如下:

    dalvik.vm.heapstartsize=8m
    dalvik.vm.heapgrowthlimit=192m
    
    #
    dalvik.vm.heapsize=512m
    
    #当前理想的堆内存利用率. GC后, Dalvik的Heap内存会进行相应的调整, 
    #调整到当前存活的对象的大小和 / Heap大小 接近这个选项的值, 
    #即这里的0.75. 注意, 这只是一个参考值.
    dalvik.vm.heaptargetutilization=0.75
    
    #单次Heap内存调整的最小值.
    dalvik.vm.heapminfree=2m
    
    #单次Heap内存调整的最大值.
    dalvik.vm.heapmaxfree=8m