------------------------------------------
#Best to isolate server and testing part 

Find the docker running once you logi as root and login to that container (docker exec -it <cont_number> /bin/bash)

exit
root@0:~# docker ps 
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                                                                                                                                       NAMES
7a76cfc6f63e   rocm      "/bin/sh -c 'jupyter…"   30 minutes ago   Up 30 minutes   0.0.0.0:8000->8000/tcp, [::]:8000->8000/tcp, 0.0.0.0:8888->8888/tcp, [::]:8888->8888/tcp, 0.0.0.0:30000->30000/tcp, [::]:30000->30000/tcp   rocm
(reverse-i-search)`exec': docker exec -it 7a7 /bin/bash 
-----------------------------------------
root@7a76cfc6f63e:/app# pip list | grep torch 
torch                             2.9.0.dev20250915+rocm7.0.2.lw.git1c57644d
torchaudio                        2.8.0+rocm7.0.2.git0c223473
torchvision                       0.24.0+rocm7.0.2.git98f8b375
root@7a76cfc6f63e:/app# pip list | grep vllm  
vllm                              0.9.2rc2.dev2602+g03b8f9b84.rocm702
root@7a76cfc6f63e:/app# 
