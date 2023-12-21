---
title: "What Is Raid"
date: 2023-12-09T11:05:56+01:00
draft: false
---

# What is RAID?
RAID stands for **R**edundant **A**rray of **I**ndependent **D**isks. And as the name says, it's about
using multiple drives in combination. The goal is often to have more speed, more reliability or even both.

## RAID Level
There are different possibilities for the combination in a RAID system. They called RAID level for different purposes.

### RAID-0
Also called Stripping. It combines two or more drives into one big space. The data is distributed over the drives.
Example: If you have 1 GB of data and two drives in RAID-0, 500 GB is written on drive one and 500 GB on drive two.
So writing and reading if is faster because it can be done at the same time.

But if one drive is failing the data can't be accessed, so the usual use case for this is when you need fast
speed and also can restore the data if an error occurs.

### RAID-1
Also called Mirroring. Like the name says, a second drive or even multiple drives will have the same content at
every time. So every write operation is performed on every drive simultanously.

If one drive is failing, the system will not fail. With RAID-1 it will be more reliable. Is often used for 
system partitions.

### RAID-5
For this setup you will need at least three drives. On two drives the data will be distributed, similiar to RAID-0. But
on the third drive a checksum of the data from the first two drives will be saved. In case of one drive
failing the data can be reconstructed. 

It has some benefits like the benefits from RAID-0 but with more reliability. So one drive can fail before 
data is lost.

### RAID-6
Here are two drives used for the checksums, like in RAID-5. So basically two drives can fail here.

### RAID-10
It's not "10" for ten. It's more a combination of RAID-0 and RAID-1, so you would say "RAID - one - zero".
Two drives are mirrored and one or more of the mirrors are used to strip them together. You can think of 
the same setup with two drives in a RAID-0 -> combining them to get more speed, but on top with a RAID-0 to 
mirror those setups.

So in the same stripset two devices can fail before data is lost.

## Different Types of RAID Systems
There are two types of RAID systems. Hardware and Software RAID. If I have the possibility to choose, I will
always go for the Hardware RAID controller, because of the spezialized Processors and Chips which make
the calculation of the parity faster than on the usual CPU.

### Software RAID
To configure a Software RAID on linux, I can use `mdadm`. For example to create a RAID 5 with three drives: 
`mdadm --create --verbose /dev/md0 --level=raid5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd`

To check if I have a RAID system installed, I can use `cat /proc/mdstat`.

After creating a RAID, I can format the new **md0** device: `mkfs -t ext4 /dev/md0`. And then mount it:
`mount /dev/md0 /mnt`.

If I restart my computer, the configuration will be gone, so I have to write the configuration for **mdadm**.
`mdadm --examine --scan >> /etc/mdadm/mdadm.conf`.



### Hardware RAID
The operation system only sees one device, which it uses as a usual drive. But at the other end there
can be a Hardware RAID or Software RAID installed with mulitple drives.


