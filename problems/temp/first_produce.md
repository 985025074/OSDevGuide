第一阶段的产物:
File: target/riscv64gc-unknown-none-elf/debug/os
Format: elf64-littleriscv
Arch: riscv64
AddressSize: 64bit
LoadName: <Not found>
ElfHeader {
  Ident {
    Magic: (7F 45 4C 46)
    Class: 64-bit (0x2)
    DataEncoding: LittleEndian (0x1)
    FileVersion: 1
    OS/ABI: SystemV (0x0)
    ABIVersion: 0
    Unused: (00 00 00 00 00 00 00)
  }
  Type: Executable (0x2)
  Machine: EM_RISCV (0xF3)
  Version: 1
  Entry: 0x0
  ProgramHeaderOffset: 0x40
  SectionHeaderOffset: 0x14F0
  Flags [ (0x5)
    EF_RISCV_FLOAT_ABI_DOUBLE (0x4)
    EF_RISCV_RVC (0x1)
  ]
  HeaderSize: 64
  ProgramHeaderEntrySize: 56
  ProgramHeaderCount: 2
  SectionHeaderEntrySize: 64
  SectionHeaderCount: 14
  StringTableSectionIndex: 12
}
Sections [
  Section {
    Index: 0
    Name:  (0)
    Type: SHT_NULL (0x0)
    Flags [ (0x0)
    ]
    Address: 0x0
    Offset: 0x0
    Size: 0
    Link: 0
    Info: 0
    AddressAlignment: 0
    EntrySize: 0
  }
  Section {
    Index: 1
    Name: .text (1)
    Type: SHT_PROGBITS (0x1)
    Flags [ (0x2)
      SHF_ALLOC (0x2)
    ]
    Address: 0x80200000
    Offset: 0x158
    Size: 0
    Link: 0
    Info: 0
    AddressAlignment: 1
    EntrySize: 0
  }
  Section {
    Index: 2
    Name: .bss (7)
    Type: SHT_NOBITS (0x8)
    Flags [ (0x3)
      SHF_ALLOC (0x2)
      SHF_WRITE (0x1)
    ]
    Address: 0x80200000
    Offset: 0x158
    Size: 0
    Link: 0
    Info: 0
    AddressAlignment: 1
    EntrySize: 0
  }
  Section {
    Index: 3
    Name: .debug_abbrev (12)
    Type: SHT_PROGBITS (0x1)
    Flags [ (0x0)
    ]
    Address: 0x0
    Offset: 0x158
    Size: 308
    Link: 0
    Info: 0
    AddressAlignment: 1
    EntrySize: 0
  }
  Section {
    Index: 4
    Name: .debug_info (26)
    Type: SHT_PROGBITS (0x1)
    Flags [ (0x0)
    ]
    Address: 0x0
    Offset: 0x28C
    Size: 1525
    Link: 0
    Info: 0
    AddressAlignment: 1
    EntrySize: 0
  }
  Section {
    Index: 5
    Name: .debug_aranges (38)
    Type: SHT_PROGBITS (0x1)
    Flags [ (0x0)
    ]
    Address: 0x0
    Offset: 0x881
    Size: 96
    Link: 0
    Info: 0
    AddressAlignment: 1
    EntrySize: 0
  }
  Section {
    Index: 6
    Name: .debug_str (53)
    Type: SHT_PROGBITS (0x1)
    Flags [ (0x30)
      SHF_MERGE (0x10)
      SHF_STRINGS (0x20)
    ]
    Address: 0x0
    Offset: 0x8E1
    Size: 1216
    Link: 0
    Info: 0
    AddressAlignment: 1
    EntrySize: 1
  }
  Section {
    Index: 7
    Name: .comment (64)
    Type: SHT_PROGBITS (0x1)
    Flags [ (0x30)
      SHF_MERGE (0x10)
      SHF_STRINGS (0x20)
    ]
    Address: 0x0
    Offset: 0xDA1
    Size: 147
    Link: 0
    Info: 0
    AddressAlignment: 1
    EntrySize: 1
  }
  Section {
    Index: 8
    Name: .riscv.attributes (73)
    Type: SHT_RISCV_ATTRIBUTES (0x70000003)
    Flags [ (0x0)
    ]
    Address: 0x0
    Offset: 0xE34
    Size: 116
    Link: 0
    Info: 0
    AddressAlignment: 1
    EntrySize: 0
  }
  Section {
    Index: 9
    Name: .debug_frame (91)
    Type: SHT_PROGBITS (0x1)
    Flags [ (0x0)
    ]
    Address: 0x0
    Offset: 0xEA8
    Size: 136
    Link: 0
    Info: 0
    AddressAlignment: 8
    EntrySize: 0
  }
  Section {
    Index: 10
    Name: .debug_line (104)
    Type: SHT_PROGBITS (0x1)
    Flags [ (0x0)
    ]
    Address: 0x0
    Offset: 0xF30
    Size: 140
    Link: 0
    Info: 0
    AddressAlignment: 1
    EntrySize: 0
  }
  Section {
    Index: 11
    Name: .symtab (116)
    Type: SHT_SYMTAB (0x2)
    Flags [ (0x0)
    ]
    Address: 0x0
    Offset: 0xFC0
    Size: 912
    Link: 13
    Info: 25
    AddressAlignment: 8
    EntrySize: 24
  }
  Section {
    Index: 12
    Name: .shstrtab (124)
    Type: SHT_STRTAB (0x3)
    Flags [ (0x0)
    ]
    Address: 0x0
    Offset: 0x1350
    Size: 142
    Link: 0
    Info: 0
    AddressAlignment: 1
    EntrySize: 0
  }
  Section {
    Index: 13
    Name: .strtab (134)
    Type: SHT_STRTAB (0x3)
    Flags [ (0x0)
    ]
    Address: 0x0
    Offset: 0x13DE
    Size: 268
    Link: 0
    Info: 0
    AddressAlignment: 1
    EntrySize: 0
  }
]
ProgramHeaders [
  ProgramHeader {
    Type: PT_GNU_STACK (0x6474E551)
    Offset: 0x0
    VirtualAddress: 0x0
    PhysicalAddress: 0x0
    FileSize: 0
    MemSize: 0
    Flags [ (0x6)
      PF_R (0x4)
      PF_W (0x2)
    ]
    Alignment: 0
  }
  ProgramHeader {
    Type: PT_RISCV_ATTRIBUTES (0x70000003)
    Offset: 0xE34
    VirtualAddress: 0x0
    PhysicalAddress: 0x0
    FileSize: 116
    MemSize: 116
    Flags [ (0x4)
      PF_R (0x4)
    ]
    Alignment: 1
  }
]
Relocations [
]
Symbols [
  Symbol {
    Name:  (0)
    Value: 0x0
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: Undefined (0x0)
  }
  Symbol {
    Name: 02ldzujiwiyum18ep6aw8qah4 (1)
    Value: 0x0
    Size: 0
    Binding: Local (0x0)
    Type: File (0x4)
    Other: 0
    Section: Absolute (0xFFF1)
  }
  Symbol {
    Name: $d (27)
    Value: 0x0
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_abbrev (0x3)
  }
  Symbol {
    Name: .L0  (30)
    Value: 0x0
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_info (0x4)
  }
  Symbol {
    Name: $d (35)
    Value: 0x0
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_info (0x4)
  }
  Symbol {
    Name: .Lline_table_start0 (38)
    Value: 0x0
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_line (0xA)
  }
  Symbol {
    Name: $d (58)
    Value: 0x0
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_aranges (0x5)
  }
  Symbol {
    Name: $d (61)
    Value: 0x2DD
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_str (0x6)
  }
  Symbol {
    Name: $d (64)
    Value: 0x5E
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .comment (0x7)
  }
  Symbol {
    Name: $d (67)
    Value: 0x0
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: Absolute (0xFFF1)
  }
  Symbol {
    Name: .L0  (70)
    Value: 0x0
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_frame (0x9)
  }
  Symbol {
    Name: $d (75)
    Value: 0x0
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_frame (0x9)
  }
  Symbol {
    Name: $d (78)
    Value: 0x0
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_line (0xA)
  }
  Symbol {
    Name: ec7onf50ki29e2qgyw2mpoes1 (81)
    Value: 0x0
    Size: 0
    Binding: Local (0x0)
    Type: File (0x4)
    Other: 0
    Section: Absolute (0xFFF1)
  }
  Symbol {
    Name: $d (107)
    Value: 0x106
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_abbrev (0x3)
  }
  Symbol {
    Name: .L0  (110)
    Value: 0x5AF
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_info (0x4)
  }
  Symbol {
    Name: $d (115)
    Value: 0x5AF
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_info (0x4)
  }
  Symbol {
    Name: .Lline_table_start0 (118)
    Value: 0x49
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_line (0xA)
  }
  Symbol {
    Name: $d (138)
    Value: 0x30
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_aranges (0x5)
  }
  Symbol {
    Name: $d (141)
    Value: 0x2DD
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_str (0x6)
  }
  Symbol {
    Name: $d (144)
    Value: 0x5E
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .comment (0x7)
  }
  Symbol {
    Name: $d (147)
    Value: 0x0
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: Absolute (0xFFF1)
  }
  Symbol {
    Name: .L0  (150)
    Value: 0x40
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_frame (0x9)
  }
  Symbol {
    Name: $d (155)
    Value: 0x40
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_frame (0x9)
  }
  Symbol {
    Name: $d (158)
    Value: 0x49
    Size: 0
    Binding: Local (0x0)
    Type: None (0x0)
    Other: 0
    Section: .debug_line (0xA)
  }
  Symbol {
    Name: BASE_ADDRESS (161)
    Value: 0x80200000
    Size: 0
    Binding: Global (0x1)
    Type: None (0x0)
    Other: 0
    Section: Absolute (0xFFF1)
  }
  Symbol {
    Name: skernel (174)
    Value: 0x80200000
    Size: 0
    Binding: Global (0x1)
    Type: None (0x0)
    Other: 0
    Section: .text (0x1)
  }
  Symbol {
    Name: stext (182)
    Value: 0x80200000
    Size: 0
    Binding: Global (0x1)
    Type: None (0x0)
    Other: 0
    Section: .text (0x1)
  }
  Symbol {
    Name: strampoline (188)
    Value: 0x80200000
    Size: 0
    Binding: Global (0x1)
    Type: None (0x0)
    Other: 0
    Section: .text (0x1)
  }
  Symbol {
    Name: etext (200)
    Value: 0x80200000
    Size: 0
    Binding: Global (0x1)
    Type: None (0x0)
    Other: 0
    Section: .text (0x1)
  }
  Symbol {
    Name: srodata (206)
    Value: 0x80200000
    Size: 0
    Binding: Global (0x1)
    Type: None (0x0)
    Other: 0
    Section: .text (0x1)
  }
  Symbol {
    Name: erodata (214)
    Value: 0x80200000
    Size: 0
    Binding: Global (0x1)
    Type: None (0x0)
    Other: 0
    Section: .text (0x1)
  }
  Symbol {
    Name: sdata (222)
    Value: 0x80200000
    Size: 0
    Binding: Global (0x1)
    Type: None (0x0)
    Other: 0
    Section: .text (0x1)
  }
  Symbol {
    Name: edata (228)
    Value: 0x80200000
    Size: 0
    Binding: Global (0x1)
    Type: None (0x0)
    Other: 0
    Section: .text (0x1)
  }
  Symbol {
    Name: sbss_with_stack (234)
    Value: 0x80200000
    Size: 0
    Binding: Global (0x1)
    Type: None (0x0)
    Other: 0
    Section: .text (0x1)
  }
  Symbol {
    Name: sbss (250)
    Value: 0x80200000
    Size: 0
    Binding: Global (0x1)
    Type: None (0x0)
    Other: 0
    Section: .bss (0x2)
  }
  Symbol {
    Name: ebss (255)
    Value: 0x80200000
    Size: 0
    Binding: Global (0x1)
    Type: None (0x0)
    Other: 0
    Section: .bss (0x2)
  }
  Symbol {
    Name: ekernel (260)
    Value: 0x80200000
    Size: 0
    Binding: Global (0x1)
    Type: None (0x0)
    Other: 0
    Section: .bss (0x2)
  }
]
VersionSymbols [
]
VersionDefinitions [
]
VersionRequirements [
]
Groups {
  There are no group sections in the file.
}
Addrsig [
]
NoteSections [
]
StackSizes [
]
