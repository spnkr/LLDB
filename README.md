# LLDB
A collection of LLDB aliases/regexes and Python scripts to aid in your debugging sessions

## Installation 

For scripts (anything ending with `.py`), you'll need to do the following: 
  1. Download scripts. Install to a dir of your choosing (i.e. `~/lldb`)
  2. In `~/.lldbinit` add the following:
      `command script import path/to/lldb_file.py`
  
  You must import each file individually in your lldbinit file

For any lldb commands simply just paste the command into your `~/.lldbinit` file


## LLDB Commands

### ls 
List a directory from the process's perspective. Useful when working on an actual device. 
```
command regex ls 's/(.+)/expression -lobjc -O -- @import Foundation; [[NSFileManager defaultManager] contentsOfDirectoryAtPath:@"%1" error:nil]/'
```
  Example: 
      
      (lldb) ls /
      (lldb) ls /System/Library
      
### reload_lldbinit
Reloads all the contents in your ~/.lldbinit file. Useful for seeing if your python script(s) broke or want to do incremental updates to a python script

```
command alias reload_lldbinit command source ~/.lldbinit
```

### tv
Toggle view. Hides/Shows a view depending on it's current state. You don't need to resume LLDB to see changes

```
command regex -- tv 's/(.+)/expression -l objc -O -- @import QuartzCore; [%1 setHidden:!(BOOL)[%1 isHidden]]; (void)[CATransaction flush];/'
```

### protocol
Dumps all the required and optional methods for specific protocol (Objective-C only)

```
command regex protocol 's/(.+)/expression -lobjc -O -- @import Foundation; NSMutableString *string = [NSMutableString string]; Protocol * prot = objc_getProtocol("%1"); [string appendFormat:@"\nProtocol: %s, %@\n", (char *)[prot name], (id)prot]; [string appendString:@"==========================================\n"]; for (int isRequired = 1; isRequired > -1; isRequired--) { [string appendFormat:@" (%@)\n", isRequired ? @"Required" : @"Optional"]; for (int isInstanceMethod = 0; isInstanceMethod < 2; isInstanceMethod++) { unsigned int ds_count = 0; struct objc_method_description * methods = (struct objc_method_description *)protocol_copyMethodDescriptionList(prot, (BOOL)isRequired, (BOOL)isInstanceMethod, &ds_count); for (int i = 0; i < ds_count; i++) { struct objc_method_description method = methods[i]; [string appendFormat:@"%@ %@, %s\n", isInstanceMethod ? @"-": @"+", NSStringFromSelector(method.name), method.types]; }}} string;/'
```

## LLDB Scripts
### dclass
Dumps all the NSObject inherited classes in the process. If you give it a module that exists on disk, it will dump only the classes within that module. You can also filter out classes to only a certain type and can also generate a header file for a specific class.
  
  Example: 
  
      # Dump ALL the NSObject classes within the process
      (lldb) dclass 

      # Dump all the classes that are a UIViewController within the process
      (lldb) dclass -f UIViewController

      # Dump all classes in CKConfettiEffect NSBundle that are UIView subclasses
      (lldb) dclass /System/Library/Messages/iMessageEffects/CKConfettiEffect.bundle/CKConfettiEffect -f UIView
      
      # Generate a header file for the class specified:
      (lldb) dclass -g UIView
      
      # Generate a protocol that you can cast an object to. Ideal when working with private classes at dev time
      (lldb) dclass -p UIView

### find
  Finds all subclasses of a class. This class must by dynamic (aka inherit from a NSObject class). Currently doesn't work with   NSString or NSNumber (tagged pointer objects). 
  
  Example: 
  
      # Find all instances and subclasses of UIView
      (lldb)  find UIView
      
      # Find all instances of UIView that are UIViews. Ignore subclasses.
      (lldb) find UIView -e
      
      #Find all instances of UIView whose tag is equal to 5. Objective-C syntax only. Can reference object by 'obj'
      (lldb) find UIView -c "[obj tag]==5"

### yoink

  Takes a path on a iOS/tvOS/watchOS and writes to the **/tmp/** dir on your computer.
  If it can be read by `-[NSData dataWithContentsOfFile:]`, it can be written to disk

  Example (on iOS 10 device): 
  
      (lldb) yoink /System/Library/Messages/iMessageEffects/CKConfettiEffect.bundle/CKConfettiEffect

![yoink example](https://github.com/DerekSelander/LLDB/raw/master/Media/yoink_gif.gif)

### pmodule

  Creates a custom dtrace script that profiles modules in an executable based upon its 
  memory layout and ASLR. Provide no arguments w/ '-a' if you want a count of all the modules firing. 
  Provide a module if you want to dump all the methods as they occur. The location of the script is 
  copied to your computer so you can paste the soon to be executed dtrace script in the Terminal. 
  
  WARNING: YOU MUST DISABLE ROOTLESS TO USE DTRACE
  
      # Trace all Objective-C code in UIKit 
      (lldb) pmodule UIKit

      # Trace all non-Objective-C code in libsystem_kernel.dylib (i.e. pid$target:libsystem_kernel.dylib::entry)
      (lldb) pmodule -n libsystem_kernel.dylib
      
      # Dump errrything. Only displays count of function calls from modules after you end the script. Warning slow
      (lldb) pmodule -a
