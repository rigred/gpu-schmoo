 
Closet Match

phils_fake_960.rom vs GPU-DB roms
Num different bytes:

106824 - 55
120365 - 55
122745 - 46
123299 - 47
124757 - 55
143074 - 2210 (disqualified)
144954 - 57
151124 - 45 (closest)
153087 - 2029 (broken mess)
156890 - 67
161520 - 47 (date is suspiciously close to fake bios)
180493 - 45
186843 - 45
192092 - 63
198103 - 33 (another fake 960)

VBIOS makes use of little endian byte order (bytes reversed)

Example:
10DE becomes DE10 in your hexeditor

Offsets:
50-5f: Strap code with AND mask
86: 2bytes Subsystem ID ()
396: 2bytes (Vendor ID)
398: 2bytes reversed Device ID?
430: 2bytes (Device ID)
26474: 1byte value (Fan Min Speed Percent)
26475: 1byte value (Fan Max Speed Percent)

58713: 2byte (Might be involved with RAM size
58718: 2bytes Device ID (AGAIN)?

Last/61439: 8bit modulus Checksum
