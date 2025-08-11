### 블로그를 쓰기 전 troubleshooting에서 내가 정리, 회고했던 글들

pc 하드웨어 Nic의 제조사가 다를때 gpu install로 커널이 업데이트되면 문제가 발생한다.
 
lspci | grep -i ethernet
 
04:00.0 Ethernet controller: Realtek Semiconductor Co., Ltd. RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller (rev 15) 
  문제가 됐던 nic의 제조사
  
00:1f.6 Ethernet controller: Intel Corporation Ethernet Connection (2) I219-V
  문제가 되지 않는 nic의 제조사 

linux가 초기에 설치되면 r8169라는 리눅스가 배포해주는 네트워크 드라이버가 설치되며 
gpu driver가 설치되면 kernel이 업데이트 됨과 동시에 해당 버전에서의 드라이버를 찾지 못해 인터넷 연결이 끊겼던 것으로 확인.

kernel을 미리 업데이트 한 후 사용하고 있는 드라이버를 설치해주면 gpu driver를 설치한 이후에도 해당 드라이버를 찾아 로딩시켜 해결되는것으로 확인.


최종 블로그

https://bluedreamer-twenty.tistory.com/8
