How to Install the selenium on windows machine 


1) Install java 
2) Install python 
	--> Set in environment variable 
		location of python installed : C:\Users\user\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Python 3.6


		Link : https://superuser.com/questions/949560/how-do-i-set-system-environment-variables-in-windows-10


	---> 	How to Check Path Variable in windows : 
			---> Go to command promt 
			---> echo %PATH%
				C:\Users\user>echo %PATH%

				"C:\ProgramData\Oracle\Java\javapath;C:\Users\user\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Python 3.6\;C:\Users\user\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Python 3.6\Scripts\";C:\WINDOWS\system32;C:\WINDOWS;C:\WINDOWS\System32\Wbem;C:\WINDOWS\System32\WindowsPowerShell\v1.0\;C:\Users\user\AppData\Local\Microsoft\WindowsApps;


			---> Present working directory in windows and in linux 
				---> In windows : cd 
				----> In linux : pwd 
				

				 Source Link :
				https://superuser.com/questions/341192/how-can-i-display-the-contents-of-an-environment-variable-from-the-command-promp


3) Install pip
running python -m pip install --upgrade pip in cmd


