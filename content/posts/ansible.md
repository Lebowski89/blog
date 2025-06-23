One of my favourite tools in my homelab setup is ansible. I love it. Upon discovering it, my setup has changed rapidly after years of much stagnancy. For example:

   - 2014-2020: I deployed containers on a Synology NAS (DS415+) and called it a day.
   - 2020-2023: I relied upon UnRaid's community applications to deploy containers. 

During these years, things were working fine, and it is when I discovered the main applications I depend on, but I wanted more. In 2023, I installed the UnRaid Compose plug-in, and moved everything to Compose files, which culminated in me moving all my Compose files to a Debian VM on UnRaid. But there was still something missing. I still had to:

  - Make sure all directories were in place
  - Ensure everything had correct permissions
  - Manually configure configs / databases / DNS records and a range of other things.

In 2024, I stumbled upon an ansible tutorial and gave it a go. My only prior experience with ansible was running Cloudbox/Saltbox on cloud servers, and with those, everything is done for you. It was apparent that ansible was what was missing in my setup. It was also apparent that ansible was a rabbit hole. It was easy to get into, and it was also easy to get things wrong, with my experiences thus far being one of much trial, much error, and ultimately, much reward. I’ve built-up, torn-down, ripped-apart and rebuilt my ansible setup to get to where I am now. Using ansible on a Ubuntu VM on Proxmox, I now deploy my containers/services easily, efficiently, automatically, with minimal to no input from myself. 

As of now, I no longer install any app or service without thinking about how to automate it with ansible. It's actually quite addictive when I think about it. Would recommend.  




