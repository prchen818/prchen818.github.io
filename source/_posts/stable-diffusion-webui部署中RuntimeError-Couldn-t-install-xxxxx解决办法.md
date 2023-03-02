---
title: stable-diffusion-webui部署中RuntimeError:Couldn't install xxxxx解决办法
date: 2023-02-23 21:57:19
tags: 
- novelai
- stable-diffusion
- debug
categories:
- debug
---

# 无法连接github

## 错误输出
```
Installing clip
Traceback (most recent call last):
  File "E:\fork\stable-diffusion-webui-master\launch.py", line 360, in <module>
    prepare_environment()
  File "E:\fork\stable-diffusion-webui-master\launch.py", line 278, in prepare_environment
    run_pip(f"install {clip_package}", "clip")
  File "E:\fork\stable-diffusion-webui-master\launch.py", line 137, in run_pip
    return run(f'"{python}" -m pip {args} --prefer-binary{index_url_line}', desc=f"Installing {desc}", errdesc=f"Couldn't install {desc}")
  File "E:\fork\stable-diffusion-webui-master\launch.py", line 105, in run
    raise RuntimeError(message)
RuntimeError: Couldn't install clip.
Command: "E:\fork\stable-diffusion-webui-master\venv\Scripts\python.exe" -m pip install git+https://github.com/openai/CLIP.git@d50d76daa670286dd6cacf3bcd80b5e4823fc8e1 --prefer-binary
Error code: 1
stdout: Collecting git+https://github.com/openai/CLIP.git@d50d76daa670286dd6cacf3bcd80b5e4823fc8e1
  Cloning https://github.com/openai/CLIP.git (to revision d50d76daa670286dd6cacf3bcd80b5e4823fc8e1) to c:\users\peifang\appdata\local\temp\pip-req-build-f_lc43m0

stderr:   Running command git clone --filter=blob:none --quiet https://github.com/openai/CLIP.git 'C:\Users\PeiFang\AppData\Local\Temp\pip-req-build-f_lc43m0'
  fatal: unable to access 'https://github.com/openai/CLIP.git/': OpenSSL SSL_read: Connection was reset, errno 10054
  error: subprocess-exited-with-error

  git clone --filter=blob:none --quiet https://github.com/openai/CLIP.git 'C:\Users\PeiFang\AppData\Local\Temp\pip-req-build-f_lc43m0' did not run successfully.
  exit code: 128

  See above for output.

  note: This error originates from a subprocess, and is likely not a problem with pip.
error: subprocess-exited-with-error

git clone --filter=blob:none --quiet https://github.com/openai/CLIP.git 'C:\Users\PeiFang\AppData\Local\Temp\pip-req-build-f_lc43m0' did not run successfully.
exit code: 128

See above for output.

note: This error originates from a subprocess, and is likely not a problem with pip.
```

## 问题原因
阅读错误输出不难发现问题出在```git clone https://github.com/xxxxxx```连接超时，显然这还是网络问题（长火城的问题）

直观的解决办法主要是三个：
1. 解决网络问题，挂代理
2. 换国内的下载源
3. 手动安装

对于我这种既懒又有些许强迫症的原教旨下载主义者（划掉）来说，当然是用办法1直接挂上梯子来的最快

但是问题就出在这了，不管我换什么梯子，走全局还是规则，亦或是走别的github代理下载，始终还是TimeOut，好像代理并未生效

实际上确实是这样，对于命令行来说普通的系统代理以及全局规则是无效的
## 解决办法
1. 使用猫猫软件的Tun大法，通过虚拟网卡的方式直接从底层接管系统流量，这样不管你是什么东西都得乖乖走代理
2. 在命令行中配置```$http_proxy```环境变量，仅在当前窗口临时生效，相当于手动为命令行设置系统代理

