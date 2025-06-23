When starting out with ansible, it's common to begin with tutorials. Good places to start are those by Jeff Geerling, Learn Linux TV, tech-tubers like Techno Tim, Jim's Garage, and a range of others. For me, these helped - but I'm a firm believer in simply getting hands on with ansible and taking a look at how others have approached their plays, tasks, roles, and then trying it out for yourself, adapting it for your needs, and building upon it. Whatever you're setting out to do, chances are someones been down the same path. Ansible Galaxy is a good source for this, but I've found [Saltbox](https://docs.saltbox.dev/) most helpful.

Saltbox is a great solution, that:
  - Automates the setting up of cloud servers in a way that is end-user friendly to new and experienced consumers alike.
  - Has a large focus towards relevant media focused apps (including the arrs)
  - Has a very large and dedicated community with a dedicated core of users working on it.

The above results in Saltbox's tasks and roles being stable, compatible, and well-tested.

So, with the above said - when sitting down to spin up a new docker stack or service, I take a look at both the Saltbox GitHub to see if the Saltbox crew have tackled something similar, to look for ideas and guidance that I can incorporate into my own roles. You will often find a good general starting point that can be built upon and adapted for your specific use-case. 

Another benefit of trying to adapt others tasks and roles to your own setup is that you'll often run into issues, obstacles, hurdles and [insert some drama-esque term here], that will have you reaching for the ansible documentation and support sites. I've found that this has forced me to actually understand what each module/task/role is trying to do, while equipping me with further tools and resources that will help me now and into the future. 

Of course, there is nothing stopping you from spinning up Saltbox on your server and calling it a day. You can even boot up Saltbox on a local machine. I am a long-time user of Saltbox and Cloudbox before that, and currently deploy it to multiple cloud servers. In my home-lab, though, I prefer more fine-grain control over my media setup, with roles built for my needs.

> [!EXAMPLE]
> Saltbox's Recyclarr role will ensure it has everything it needs to install and run with an empty config. You then need to go through and edit the config with your desired settings. If you build upon this with your own role, you can fully-automate it and have all your desired config and secrets settings in place and ready to go, such as radarr:

```
- name: Insert radarr config settings
  ansible.builtin.blockinfile:
    path: '{{ recyclarr_defaults_location }}/configs/radarr.yml'
    marker: "## {mark} ANSIBLE RADARR MANAGED BLOCK ##"
    owner: '{{ puid }}'
    group: '{{ pgid }}'
    mode: '0664'
    create: true
    block: |
      radarr:
        radarr:
          media_naming:
            folder: plex
            movie:
              rename: true
              standard: default
          quality_definition:
            type: movie
          quality_profiles:
            - name: Remux + WEB 1080p
              reset_unmatched_scores:
                enabled: true
              upgrade:
                allowed: true
                until_quality: Remux-1080p
                until_score: 10000
              min_format_score: 0
              quality_sort: top
              qualities:
                - name: Remux-1080p
                - name: Bluray-1080p
                  enabled: false
                - name: WEB 1080p
                  qualities:
                    - WEBRip-1080p
                    - WEBDL-1080p
          delete_old_custom_formats: true
          replace_existing_custom_formats: true
          custom_formats:
            - trash_ids:
                  # Audio Advanced #1
                - 417804f7f2c4308c1f4c5d380d4c4475 # ATMOS (undefined)
                - 1af239278386be2919e1bcee0bde047e # DD+ ATMOS
                - 185f1dd7264c4562b9022d963ac37424 # DD+
                - f9f847ac70a0af62ea4a08280b859636 # DTS-ES
                - dcf3ec6938fa32445f590a4da84256cd # DTS-HD MA
                - 2f22d89048b01681dde8afe203bf2e95 # DTS X
                - 1c1a4c5e823891c75bc50380a6866f73 # DTS
                - 496f355514737f7d83bf7aa4d24f8169 # TrueHD ATMOS
                - 3cafb66171b47f226146a0770576870f # TrueHD
                  # Audio Advanced #2
                - 240770601cc226190c367ef59aba7463 # AAC
                - c2998bd0d90ed5621d8df281e839436e # DD
                - 8e109e50e0a0b83a5098b056e13bf6db # DTS-HD HRA
                - a570d4a0e56a2874b64e5bfa55202a1b # FLAC
                - 6ba9033150e7896bdc9ec4b44f2b230f # MP3
                - a061e2e700f81932daf888599f8a8273 # Opus
                - e7c2fcae07cbada050a0af3357491d7b # PCM
                  # HDR Formats
                - e23edd2482476e595fb990b12e7c609c # DV HDR10
                - 55d53828b9d81cbe20b02efd00aa0efd # DV HLG
                - a3e19f8f627608af0211acd02bf89735 # DV SDR
                - 58d6a88f13e2db7f5059c41047876f00 # DV
                - 2a4d9069cc1fe3242ff9bdaebed239bb # HDR (undefined)
                - e61e28db95d22bedcadf030b8f156d96 # HDR
                - dfb86d5941bc9075d6af23b09c2aeecd # HDR10
                - b974a6cd08c1066250f1f177d7aa1225 # HDR10+
                - 9364dd386c9b4a1100dde8264690add7 # HLG
                - 08d6d8834ad9ec87b1dc7ec8148e7a1f # PQ
                  # HQ Release Groups
                - 3a3ff47579026e76d6504ebea39390de # Remux Tier 01
                - 9f98181fe5a3fbeb0cc29340da2a468a # Remux Tier 02
                - 8baaf0b3142bf4d94c42a724f034e27a # Remux Tier 03
                - 403816d65392c79236dcb6dd591aeda4 # WEB Tier 02
                - c20f169ef63c5f40c2def54abaf4438e # WEB Tier 01
                - af94e0fe497124d1f9ce732069ec8c3b # WEB Tier 03
                  # Movie Versions
                - eca37840c13c6ef2dd0262b141a5482f # 4K Remaster
                - e0c07d59beb37348e975a930d5e50319 # Criterion Collection
                - 0f12c086e289cf966fa5948eac571f44 # Hybrid
                - 9f6cbff8cfe4ebbc1bde14c7b7bec0de # IMAX Enhanced
                - eecf3a857724171f968a66cb5719e152 # IMAX
                - 570bc9ebecd92723d2d21500f4be314c # Remaster
                - 957d0f44b592285f26449575e8b1167e # Special Edition
                - e9001909a4c88013a359d0b9920d7bea # Theatrical Cut
                - 9d27d9d2181838f76dee150882bdc58c # Masters of Cinema
                - 09d9dd29a0fc958f9796e65c2a8864b4 # Open Matte
                - db9b4c4b53d312a3ca5f1378f6440fc9 # Vinegar Syndrome
                  # Unwanted
                - cae4ca30163749b891686f95532519bd # AV1
                - ed38b889b31be83fda192888e2286d83 # BR-DISK
                - e6886871085226c3da1830830146846c # Generated Dynamic HDR
                - 90a6f9a284dff5103f6346090e6280c8 # LQ
                - e204b80c87be9497a8a6eaff48f72905 # LQ (Release Title)
                - 712d74cd88bceb883ee32f773656b1f5 # Sing-Along Versions
                - b8cd450cbfa689c0259a01d9e29ba3d6 # 3D
                - dc98083864ea246d05a42df0d05f81cc # x265 (HD)
                - bfd8eb01832d646a0a89c4deb46f8564 # Upscaled
                - 0a3f082873eb454bde444150b70253cc # Extras
                  # Misc
                - b6832f586342ef70d9c128d40c07b872 # Bad Dual Groups
                - ae9b7c9ebde1f3bd336a8cbd1ec4c5e5 # No-RlsGroup
                - 5c44f52a8714fdd79bb4d98e2673be1f # Retags
                - 182fa1c42a2468f8488e6dcf75a81b81 # Internal
                - c465ccc73923871b3eb1802042331306 # Line/Mic Dubbed
                - e7718d7a3ce595f289bfee26adc178f5 # Repack/Proper
                - ae43b294509409a6a13919dedd4764c4 # Repack2
                - 5caaaa1c08c1742aa4342d8c4cc463f2 # Repack3
                - 2899d84dc9372de3408e6d8cc18e9666 # x264
                - 0d91270a7255a1e388fa85e959f359d8 # FreeLeech
                  # Streaming Services
                - b3b3a6ac74ecbd56bcdbefa4799fb9df # AMZN
                - 40e9380490e748672c2522eaaeb692f7 # ATVP
                - 84272245b2988854bfb76a16e60baea5 # DSNP
                - 509e5f41146e278f9eab1ddaceb34515 # HBO
                - 5763d1b0ce84aff3b21038eea8e9b8ad # HMAX
                - 526d445d4c16214309f0fd2b3be18a89 # Hulu
                - e0ec9672be6cac914ffad34a6b077209 # iT
                - 6a061313d22e51e0f25b7cd4dc065233 # MAX
                - 170b1d363bd8516fbf3a3eb05d4faff6 # NF
                - c9fd353f8f5f1baf56dc601c4cb29920 # PCOK
                - e36a0ba1bc902b26ee40818a1d59b8bd # PMTP
                - c2863d2a50c9acad1fb50e53ece60817 # STAN
              assign_scores_to:
                - name: Remux + WEB 1080p
```

> [!NOTE]
> The above example is not a slight against Sandbox, as it needs to be accessible to a large user-base with diverse needs. The example simply highlights the value of building your own roles that are catered for your specific needs. Heck, you can even spin up your own roles and containers on an existing Saltbox install. I do this with cross-seed.

<a href="https://buymeacoffee.com/lebowski89" target="_blank"><img src="https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png" alt="Buy Me A Coffee" style="height: 41px !important;width: 174px !important;box-shadow: 0px 3px 2px 0px rgba(190, 190, 190, 0.5) !important;-webkit-box-shadow: 0px 3px 2px 0px rgba(190, 190, 190, 0.5) !important;" ></a>
