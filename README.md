# SANS Holiday Hack Challenge 2019 Xmas Cheer Laser Challenge Writeup
2019.kringlecon.com

![image](https://user-images.githubusercontent.com/54376366/70858214-19df3e80-1ebb-11ea-8b3f-c229510aa309.png)

We are greeted with a message of the day, focusing on `/home/callingcard.txt` and `(Invoke-WebRequest -Uri http://localhost:1225/).RawContent`.

Invoking localhost displays 7 possible API calls to the laser. 4 calls are to modify the following settings: angle, temperature, gas, and refraction.

![image](https://user-images.githubusercontent.com/54376366/70858279-7727bf80-1ebc-11ea-8ea2-f0ba040596fb.png)

These Web API calls shown above will be used once all settings are discovered.

Viewing callingcard.txt points us to the history. PowerShell automatically maintains a history of each session. 
![image](https://user-images.githubusercontent.com/54376366/70858330-5a3fbc00-1ebd-11ea-9acb-475440b49257.png)

Get-History will reveal our first setting and the next clue.
![image](https://user-images.githubusercontent.com/54376366/70858379-5eb8a480-1ebe-11ea-9501-a8e2b9acf9ab.png)

**Setting #1**: Angle = 65.5

That last command in history needs to be expanded: `(history).CommandLine[8]`.

"I have many name=value variables that I share to applications system wide. At a command I will reveal my secrets once you Get my Child Items."


"Applications system wide" is a lead to the environment variables. Tab completion on $env: shows the `$env:riddle` variable.

![image](https://user-images.githubusercontent.com/54376366/70858555-3aaa9280-1ec1-11ea-8ed1-bef2b2675f53.png)

Based on the riddle, it is clear how we find the next file.
```sh
Get-ChildItem /etc -Recurse | Sort LastWriteTime | Select -Last 1
```
![image](https://user-images.githubusercontent.com/54376366/70858647-0d5ee400-1ec3-11ea-97db-077a7eddfab7.png)

This returned a zipped file named archive. To unzip, run: 
```sh
Expand-Archive -Path /etc/apt/archive -DestinationPath (pwd)
```
A folder named refraction now exists in the working directory and contains a text file named riddle and an ELF file named runme. Getting the content of riddle reads:

"Very shallow am I in the depths of your elf home. You can find my entity by using my md5 identity: 25520151A320B5B0D21561F92C8F6224"


The /home/elf/depths directory contains many txt files whose md5 hashes could match the above. Here we can loop through the txt files and compare hashes to reveal **Setting #2**: Temperature = -33.5 and the next clue.
 ```sh
$files = Get-childitem /home/elf/depths -recurse | Where {$_.name -match '.txt'}
foreach ($file in $files) {
    if ((Get-FileHash $file -algorithm md5).hash -match'25520151A320B5B0D21561F92C8F6224' ) {
        Get-Content $file
    }
}
```

"I am one of many thousand similar txt's contained within the deepest of /home/elf/depths. Finding me will give you the most strength but doing so will require Piping all the FullName's to Sort Length."


Running the following commands displays the deepest directory.
```sh
$folders = gci ./depths/ -Recurse -Directory                                        
$count=0
$depths = foreach ($folder in $folders) {
    $t=($folder.fullname.split('/')).count;
    if ($t -gt $count) {
        $count=$t; $folder.fullname
    }
}
$depths = $depths | Select -Last 1 | Get-Childitem
$depths
```
This will reveal the file 0jhj5xz6.txt which takes us to our next clue.
![image](https://user-images.githubusercontent.com/54376366/70858928-261dc880-1ec8-11ea-9665-d2aaf753b896.png)

We will have to stop processes based on Usernames in the order displayed above.
```sh
'bushy','alabaster','minty','holly' | Foreach {
    $p = $_
    Stop-Process (Get-Process -IncludeUsername | 
    Where {$_.Username -eq $p}).Id
}
```

This creates another clue: `/shall/see`. Getting content from see reads:

"Get the .xml children of /etc - an event log to be found. Group all .Id's and the last thing will be in the Properties of the lonely unique event Id."

First we need to get the xml file.
```sh
$xmlFile = (Get-Childitem /etc -Recurse | Where {$_.name -match '.xml'})
```
Now run the following command to get the lonely Id.
```sh
Import-Clixml $xmlFile | select id | group id
```
![image](https://user-images.githubusercontent.com/54376366/70859011-ca543f00-1ec9-11ea-9403-43da88738b81.png)

Get properties of Id 1.
```sh
Import-Clixml $xmlFile | Where {$_.ID -eq 1}
```
![image](https://user-images.githubusercontent.com/54376366/70859028-25863180-1eca-11ea-89aa-ad7e726aff02.png)

The Message reveals **Setting 3**: gases = O=6&H=7&He=3&N=4&Ne=22&Ar=11&Xe=10&F=20&Kr=8&Rn=9

Our final setting, refraction, will be found in the refraction folder. runme.elf needs to be executed by setting its permission to execute.
```sh
chmod +x ./runme.elf
./runme.elf
```

This reveals the fourth and final **Setting #4**: refraction = 1.867


Now that we have all settings to power the laser with, we can start making api calls.
Before any settings are modified, a restart of the laser is required.
```sh
Invoke-RestMethod http://localhost:1225/api/off -Method Get
Invoke-RestMethod http://localhost:1225/api/on -Method Get
```

Now to apply the settings and **complete** this CTf.
```sh
irm http://localhost:1225/api/angle?val=65.5 -Method Get
irm http://localhost:1225/api/temperature?val=-33.5 -Method Get
irm http://localhost:1225/api/refraction?val=1.867 -Method Get
irm http://localhost:1225/api/gas -Method Post -Body "O=6&H=7&He=3&N=4&Ne=22&Ar=11&Xe=10&F=20&Kr=8&Rn=9"
irm http://localhost:1225/api/output
```


![image](https://user-images.githubusercontent.com/54376366/70859238-ccb89800-1ecd-11ea-9729-c4011b559819.png)
