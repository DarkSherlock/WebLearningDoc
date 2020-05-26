+ start a specified Activity: adb shell am start -n {processName}/{activityPath) -d {data_content}  
  processName is ApplicationId.  
+ dump memory info : adb shell dumpsys meminfo -a {processName}(or pid)
+ view activities info: adb -s emulator-5556 activity activities
+ dump heap info: adb -s emulator-5554 shell am dumpheap {processName}  /data/local/tmp/rooms.hprof  
  then pull rooms.hprof into pc
+ view application's version info: adb shell dumpsys package {processName}
