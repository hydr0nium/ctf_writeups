# Matrix
In this challenge we need to connect to a network endpoint and "crack" the coding challenge of deadface to get the flag.

>Turbo Tactical is looking to infiltrate DEADFACE. We know they're picky about whom they allow into their group, and recently they've started vetting new members. Our infiltrator needs your help solving a challenge on one of their remote servers. Check out Ghost Town for more information about what DEADFACE is looking for.
>
>code.deadface.io:50000
>
>Submit the flag as flag{flag_goes_here}.

After searching a bit on the ghosttown forum we can find this post:

[Vetting New Recruits](https://ghosttown.deadface.io/t/vetting-new-recruits/62)

where they talk about how they will recruit new deadface members. The important part is the this:
>Sounds good, man. There’s this matrix programming challenge I saw a while ago that would be a good fit. Basically, players are given a 4x5 matrix (5 lists, 4 elements each) and they have to get the sum of the smallest number in each row.

Part of the source code is also linked and can be found here:

[Source Code](https://pastebin.com/qmCpWP2p)

Let's start by connecting to the endpoint and see what we get:
```bash
$ nc code.deadface.io 50000                                       
[925780, 691216, 113708, 361275]
[527235, 452648, 469958, 155366]
[457906, 341572, 646817, 680289]
[159937, 671505, 308142, 750046]
[748104, 130423, 171242, 574649]
Too slow! Matrix reset.
```

When we look into the source code we see exactly what we should be seeing. Nice
Lets write some python code to solve that challenge. I first tried using pwntools but I could get it to work for some reason. Talking to syyntax about it, it seemed like other people had problems with pwntools as well. Then we do it the old way using sockets:
```py
import socket

response = ""
while "flag" not in response: # Because sometimes python was too slow. Just making sure it would work.
    host = "code.deadface.io"
    port = 50000
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((host,port))
        print("[#] SOCKET CONNECTED")
        all = s.recv(8192).decode().split("\n")[:-1] # Receive everything split it at \n and remove the last element because it is empty 
        for i in range(5):
            all[i] = eval(all[i]) # Running eval on all the strings to convert them to lists
        # Making it easy to read:
        line1 = all[0]
        line2 = all[1]
        line3 = all[2]
        line4 = all[3]
        line5 = all[4]
        mind = str(min(line1)+min(line2)+min(line3)+min(line4)+min(line5)).encode() # Doing what is decribed on the ghosttown forum
        print(mind.decode())
        s.sendall(mind)
        response = s.recv(8192).decode()
        print("[#] FLAG:",response) # Getting the flag
        s.close()
```
Running the code and we get the flag. Nice
<details>
<summary>Flag</summary>

```
flag{j4cked_int0_th3_matr1x}
```

</details>
