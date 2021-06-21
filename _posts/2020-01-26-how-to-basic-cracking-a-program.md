---
layout: post
title:  "Basic cracking a program using Cutter"
date:   2020-01-25 14.30:00 +0700
categories: linux
author: xb4dc0d3
tags: Reverse-Engineering,Security,Cracking
---

{: style="text-align:center"}
![useful image]({{ site.url }}/assets/images/radare-image-logo.png){:class="img-responsive}

1. Install Cutter from this link <a target="_blank" href="https://github.com/radareorg/cutter">Cutter Radare</a>.

2. Create a simple program called `vulnerable_license.c`.
   
   ```cpp
    #include <string.h>
    #include <stdio.h>
        
    int main(int argc, char *argv[]) {
        if(argc==2) {
                printf("Checking License: %s\n", argv[1]);
                if(strcmp(argv[1], "AABB-CCDD-20-OK")==0) {
                        printf("Access Granted!\n");
                } else {
                        printf("WRONG!\n");
                }
            } 
        else {
                printf("Usage: <key>\n");
        }
        return 0;
    }
  ```
    This program basically check the correct license to get the access of the program.

3. Then, compile the program using default `gcc` option.
   ```bash
   gcc vulnerable_license.c -o vulnerable_license
   ```

    ```bash
    Example: if your input is "AABB-CCDD-20-OK" you get the access, else you get "WRONG!".

    $ ./vulnerable_license AABB-CCDD-21-MM 
    Checking License: AABB-CCDD-21-MM
    WRONG!


    $ ./vulnerable_license AABB-CCDD-20-OK 
    Checking License: AABB-CCDD-20-OK
    Access Granted!
    ```

4. Open your `Cutter` app and load the compiled file before and make sure to checked the write mode.
   
   {: style="text-align:center"}
   ![useful image]({{ site.url }}/assets/images/cutter-app.png){:class="img-responsive}

   {: style="text-align:center"}
   ![useful image]({{ site.url }}/assets/images/load-write-mode-cutter.png){:class="img-responsive}
   
5. You will look something like this.
   {: style="text-align:center"}
   ![useful image]({{ site.url }}/assets/images/all-vuln-license.png){:class="img-responsive}

   We interested in `main` function, because we see there's a checker, so we need to bypass the checker by changing the flow of the program.

   {: style="text-align:center"}
   ![useful image]({{ site.url }}/assets/images/vulnerable-license.png){:class="img-responsive}
    
   From the picture above, there's some mechanism to change the flow by using `reverse jump` in Cutter. We tend to change the flow of result `test eax eax`, by default if the result not `zero` or `the string is not same` it will jump to "WRONG!" so we need to reverse the logic, by allowing the `wrong string` to grant the access.

6. You need to click the `jne 0x11b9` in my case, other computers maybe different. `Edit -> Reverse Jump`.
   
   The new change will similar like this.

   {: style="text-align:center"}
   ![useful image]({{ site.url }}/assets/images/access-license.png){:class="img-responsive}

   By runtime the flow of the program is changed.
   
7. Save the changes and try to run the program again.
   ```bash
    $ ./vulnerable_license AABB-CCDD-21-MM 
    Checking License: AABB-CCDD-21-MM
    Access Granted!

    $ ./vulnerable_license AABB-CCDD-20-OK
    Checking License: AABB-CCDD-20-OK
    WRONG!
   ```

Source: <a target="_blank" href="https://www.megabeets.net/5-ways-to-patch-binaries-with-cutter/">Patch Binaries with Cutter</a>
