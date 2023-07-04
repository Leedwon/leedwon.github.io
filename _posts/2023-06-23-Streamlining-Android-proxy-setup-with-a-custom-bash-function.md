---
title: Streamlining Android proxy setup with a custom bash function
date: 2023-06-23
categories: [Android]
tags: [android]
img_path: /assets/img/
---

As an Android developer or tester, you have likely had to deal with the challenges of intercepting network traffic for an Android device. Tools like [Charles](https://www.charlesproxy.com/) or [Proxyman](https://proxyman.io/) can be very useful in these scenarios, providing an intuitive way to monitor and manipulate the network requests and responses. But before these tools can work their magic, you first have to set up the proxy settings on your Android device or emulator, which can be an annoying process.

The traditional approach involves manually entering the Proxy settings on your device, or remembering the adb command `adb shell settings put global http_proxy 192.168.1.111:8080`. Although these methods work, there's room for improvement.

![setting proxy manually on the device](setting_proxy_manually.png)

## Introducing: A custom `adb` Bash Function

```bash
function adb() {
  if [[ $1 == "setProxy" ]]; then
    if [[ $# -eq 2 ]]; then
      command adb shell settings put global http_proxy "$2"
      echo "Proxy set to $2"
    else
      # Determine the operating system
      local os=$(uname)
      local host_ip=""

      # Depending on the OS, use the appropriate command to retrieve the local IP address
      case $os in
        "Linux")  host_ip=$(hostname -I | cut -d' ' -f1) ;;
        "Darwin") host_ip=$(ipconfig getifaddr en0)
                  [[ -z "$host_ip" ]] && host_ip=$(ipconfig getifaddr en1) ;;
        "CYGWIN"|"MINGW"|"MSYS") host_ip=$(ipconfig | grep -o 'IPv4[^:]*:[^0-9]*\([0-9]\{1,3\}\.\)\{3\}[0-9]\{1,3\}' | sed 's/[^0-9\.]*//g' | head -n1) ;;
      esac
      command adb shell settings put global http_proxy "$host_ip:9090"
      echo "Proxy set to $host_ip:9090"
    fi
  elif [[ $1 == "resetProxy" ]]; then
    command adb shell settings put global http_proxy :0
    echo "Proxy resetted"
  else
    command adb "$@"
  fi
}
```

This custom bash function simplifies the process by enhancing the basic adb command with two additional operations: `setProxy` and `resetProxy`.

The `setProxy` command allows you to set up a proxy quickly. By passing an argument in the format ip:port, you can set the proxy to the specified IP address and port. For instance, to set up a proxy, you can type: `adb setProxy 192.168.1.111:8080`.

Moreover if you can't remember the local IP address of your machine, don't worry - this function has you covered. It automatically retrieves your local IP and sets it as the proxy with `9090` port as default. To use this feature simple call `adb setProxy` without any arguments.

The `resetProxy` operation resets the proxy settings.

If the function receives arguments other than `setProxy` or `resetProxy`, it passes those arguments directly to the standard adb command, ensuring that this function can be a direct replacement for adb without affecting other operations.

To incorporate this function into your workflow, copy it into your .zshrc file and source the file.

If you'd like to use a different port instead of the default 9090, simply replace it with your preferred port in lines 18 and 19:

```bash
command adb shell settings put global http_proxy "$host_ip:your_port_goes_here"
echo "Proxy set to $host_ip:your_port_goes_here"
```

I hope you find this useful, Happy Coding!

## Resources

- [Gist with  the code](https://gist.github.com/Leedwon/fbebd3701b1b17c920d0be9c93d0a3cf)
