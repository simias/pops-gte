# Introduction #

I was bored. Here is some reversing of MDEC emulation from Pops.

## MDEC Control Block ##

Important variables used in mdec emulation.

```
0x00119B70 - mdec struct
+0      : reg0
+4
+8      : block_size
+12
+16
+20     : block_ptr
+24
```

## MDEC Init ##

Called once from main()->PSXInit() call.

```
F00C:
void InstallMDECHooks (void)
{
    InstallHardwareHooks ( 0x1F801820, 8, MDECReadRegister, MDECWriteRegister );
}
```

## Register Access ##

```
#define MDEC1_BUSY          0x20000000
#define MDEC1_RESET         0x80000000

F1D0:
u32 MDECReadRegister (u32 addr, int read_type)
{
    [0x00010214] += 4;      // clock counter ?

    if ( addr & 0xf )
    {
        if ( [0x00119B70 + 12] ) busy = MDEC1_BUSY;
        else busy = 0;
        a0 = mdec.reg0 >> 25;
        return busy | ( (a0 & 0xf) << 23 );
    }
    else return 0;
}

F21C:
void MDECWriteRegister (u32 addr, u32 value, int write_type)
{
    if ( addr & 0xf )
    {
        if ( value & MDEC1_RESET ) MDECReset ();
    }
    else mdec.reg0 = value;
}

```

## MDEC Reset ##

Called from MDECWriteRegister and from PSXReset() routine (sub\_03F8).

```
EFB4:
void MDECReset (void)
{
    // clear mdec control block
    [0x00119B70 + 24] = 0
    [0x00119B70 + 4 ] = 0
    mdec.block_size = 0
    [0x00119B70 + 12] = 0
    [0x00119B70 + 16] = 0
    mdec.block_ptr = 0
    mdec.reg0 = 0

    SetDMAHandler ( 0, MDEC_DMA0 );
    SetDMAHandler ( 1, MDEC_DMA1 );
}

```

## DMA Handling ##

```
#define MDEC0_SIZE_MASK     0x0000FFFF

    float   iqtab[64*2];        // 0 ... 2 ... 4 ... 6  - Y component
                                //... 1 ... 2 ... 5     - UV component

    u16 iqscale[64] = {
        8000 B18B  B18B A73D  F642 A73D  E7F8 9683   
        E7F8 9683  8000 D0C4  DA82 D0C4  6492 8000   
        B18B C4A7  C4A7 B18B  6492 4546  8B7E A73D  
        B0FC A73D  8B7E 4546  2351 6016  8366 9683  
        9683 8366  6016 2351  30FC 5A82  7642 8000  
        7642 5A82  30FC 2E24  5175 6492  6492 5175  
        2E24 2987  4546 4F04  4546 2987  2351 366D  
        366D 2351  1BBF 257E  1BBF 131D  131D 09BE  
    }

F040:
int MDEC_DMA0 (u32 addr, int length )
{
    addr &= 0x1FFFFFFF;
    u8 * ramptr = 0x09800000 | addr;
    if (addr > 0x007FFFFF) ramptr = NULL;

    switch ( mdec.reg0 >> 29 )
    {
        case 0:
            return length;

        case 1:     // decode
            mdec.block_size = (mdec.reg0 & MDEC0_SIZE_MASK) * 2;
            mdec.block_ptr = ramptr;
            if ([0x00119B70 + 16])
            {
                [0x00119B70 + 12] = length;
                MDECDecode ();
            }
            sub_00017B4C ();
            return 0;

        case 2:     // quantization table upload
            for (int i=0; i<128; i++)
            {
                float iq;
                int prod = ramptr[i] * iqscale[i & 0x3f];
                if ( (i & 0x3F) == 0 ) prod = prod * 8;
                if (i & 0x40) iq = (float)prod * 1.1368683772161603e-13f;   // 0x2A000000 as hex
                else iq = (float)prod * 1.7719999551773071f;    // 0x3FE2D0E5 as hex
                iqtab[(i & 0x3F) * 2 + i / 64] = iq;
            }
            return length;
    }
}

======================================================================

F164:
int MDEC_DMA1 (u32 addr, int length )
{
    addr &= 0x1FFFFFFF;
    u8 * ramptr = 0x09800000 | addr;
    if (addr > 0x007FFFFF) ramptr = NULL;

    int old_size = mdec.block_size;
    mdec.block_size = length;
    if ( old_size > 0 )
    { 
        if ( [0x00119B70 + 12] ) MDECDecode ();
    }
    else mdec.block_ptr = ramptr;
    return 0;
}

// Small procedure, setting some unknown flag.
u8 _0x002092FF;

void sub_00017B4C ()
{
    _0x002092FF = 1;
}
```

## MDEC Decode ##

WHOA this procedure is huge + extensive VFPU optimizations.. So maybe later :=)

```
F258:
void MDECDecode (void)
{
}


sub_0000F258:       ; Refs: 0x0000F154 0x0000F1C0 
    0x0000F258: 0x27BDFFD0 '...'' - addiu      $sp, $sp, -48
    0x0000F25C: 0xAFB10014 '....' - sw         $s1, 20($sp)
    0x0000F260: 0x3C110012 '...<' - lui        $s1, 0x12
; Data ref 0x00119B70 ... 0x00000000 0x00000000 0x00000000 0x00000000 
    0x0000F264: 0x26279B70 'p.'&' - addiu      $a3, $s1, -25744
    0x0000F268: 0xAFBF002C ',...' - sw         $ra, 44($sp)
    0x0000F26C: 0xAFBC0028 '(...' - sw         $gp, 40($sp)
    0x0000F270: 0xAFB50024 '$...' - sw         $s5, 36($sp)
    0x0000F274: 0xAFB40020 ' ...' - sw         $s4, 32($sp)
    0x0000F278: 0xAFB3001C '....' - sw         $s3, 28($sp)
    0x0000F27C: 0xAFB20018 '....' - sw         $s2, 24($sp)
    0x0000F280: 0xAFB00010 '....' - sw         $s0, 16($sp)
    0x0000F284: 0x8CE9000C '....' - lw         $t1, 12($a3)
    0x0000F288: 0x8CE50008 '....' - lw         $a1, 8($a3)
    0x0000F28C: 0x8CF00014 '....' - lw         $s0, 20($a3)
    0x0000F290: 0x000947C2 '.G..' - srl        $t0, $t1, 31
    0x0000F294: 0x01283021 '!0(.' - addu       $a2, $t1, $t0
    0x0000F298: 0x00061843 'C...' - sra        $v1, $a2, 1
    0x0000F29C: 0x8CE80018 '....' - lw         $t0, 24($a3)
    0x0000F2A0: 0x8CE60010 '....' - lw         $a2, 16($a3)
    0x0000F2A4: 0x00A3202D '- ..' - min        $a0, $a1, $v1
    0x0000F2A8: 0x00041040 '@...' - sll        $v0, $a0, 1
    0x0000F2AC: 0x02026021 '!`..' - addu       $t4, $s0, $v0
    0x0000F2B0: 0x02004821 '!H..' - move       $t1, $s0
    0x0000F2B4: 0x01003821 '!8..' - move       $a3, $t0
    0x0000F2B8: 0x10800292 '....' - beqz       $a0, loc_0000FD04
    0x0000F2BC: 0x0106C021 '!...' - addu       $t8, $t0, $a2
    0x0000F2C0: 0x3404FE00 '...4' - li         $a0, 0xFE00
    0x0000F2C4: 0x952A0000 '..*.' - lhu        $t2, 0($t1)

loc_0000F2C8:       ; Refs: 0x0000F2D8 
    0x0000F2C8: 0x15440005 '..D.' - bne        $t2, $a0, loc_0000F2E0
    0x0000F2CC: 0x012C182B '+.,.' - sltu       $v1, $t1, $t4
    0x0000F2D0: 0x25290002 '..)%' - addiu      $t1, $t1, 2
    0x0000F2D4: 0x012C182B '+.,.' - sltu       $v1, $t1, $t4
    0x0000F2D8: 0x5460FFFB '..`T' - bnezl      $v1, loc_0000F2C8
    0x0000F2DC: 0x952A0000 '..*.' - lhu        $t2, 0($t1)

loc_0000F2E0:       ; Refs: 0x0000F2C8 
    0x0000F2E0: 0x3C010003 '...<' - lui        $at, 0x3
; Data ref 0x0002D4F1 ... 0x9D 0x46 0x3E 0xB4 0x6D 0x02 0x3F 0xD8 0x8B 0x4A 0x3F 0x00 0x00 0x00 0x00 0x00 
    0x0000F2E4: 0xD830D4F1 '..0.' - lv.q       R400, -11024($at)
; Data ref 0x0002D4E1 ... 0x8B 0x8A 0x3F 0xF3 0x04 0xB5 0x3F 0x5E 0x83 0xEC 0x3F 0x75 0x3D 0x27 0x40 0xE0 
    0x0000F2E8: 0xD833D4E1 '..3.' - lv.q       R403, -11040($at)
    0x0000F2EC: 0xEBAF0000 '....' - sv.s       S330, 0($sp)
    0x0000F2F0: 0x0118582B '+X..' - sltu       $t3, $t0, $t8
    0x0000F2F4: 0x006B2024 '$ k.' - and        $a0, $v1, $t3
    0x0000F2F8: 0x1080026E 'n...' - beqz       $a0, loc_0000FCB4
    0x0000F2FC: 0x3C130003 '...<' - lui        $s3, 0x3
; Data ref 0x0002D588 ... 0x43040000 0x43000000 0x3F4A8BD8 0x20726F66 
    0x0000F300: 0xC664D588 '..d.' - lwc1       $fpr04, -10872($s3)
    0x0000F304: 0x3C0F0003 '...<' - lui        $t7, 0x3
    0x0000F308: 0x3C0E0003 '...<' - lui        $t6, 0x3
    0x0000F30C: 0x3C0D0001 '...<' - lui        $t5, 0x1
    0x0000F310: 0x3C120001 '...<' - lui        $s2, 0x1
; Data ref 0x0002D58C ... 0x43000000 0x3F4A8BD8 0x20726F66 0x6170614A 
    0x0000F314: 0xC5E3D58C '....' - lwc1       $fpr03, -10868($t7)
; Data ref 0x0002D590 ... 0x3F4A8BD8 0x20726F66 0x6170614A 0x0000006E 
    0x0000F318: 0xC5C2D590 '....' - lwc1       $fpr02, -10864($t6)
    0x0000F31C: 0x35B93C00 '.<.5' - ori        $t9, $t5, 0x3C00
    0x0000F320: 0x3C0E0800 '...<' - lui        $t6, 0x800
    0x0000F324: 0x240DFFFA '...$' - li         $t5, -6
    0x0000F328: 0x364F3600 '.6O6' - ori        $t7, $s2, 0x3600
    0x0000F32C: 0x2413FFFA '...$' - li         $s3, -6

loc_0000F330:       ; Refs: 0x0000FCAC 
    0x0000F330: 0x0013E200 '....' - sll        $gp, $s3, 8

loc_0000F334:       ; Refs: 0x0000FC84 
    0x0000F334: 0x2A65FFFC '..e*' - slti       $a1, $s3, -4
    0x0000F338: 0x3C150001 '...<' - lui        $s5, 0x1
    0x0000F33C: 0x03999021 '!...' - addu       $s2, $gp, $t9
    0x0000F340: 0x38A40001 '...8' - xori       $a0, $a1, 0x1
    0x0000F344: 0x36B43800 '.8.6' - ori        $s4, $s5, 0x3800
    0x0000F348: 0x0284900B '....' - movn       $s2, $s4, $a0
    0x0000F34C: 0x3403FE00 '...4' - li         $v1, 0xFE00

loc_0000F350:       ; Refs: 0x0000F354 
    0x0000F350: 0x95220000 '..".' - lhu        $v0, 0($t1)
    0x0000F354: 0x1043FFFE '..C.' - beq        $v0, $v1, loc_0000F350
    0x0000F358: 0x25290002 '..)%' - addiu      $t1, $t1, 2
    0x0000F35C: 0xF3868080 '....' - vmzero.q   M000
    0x0000F360: 0x00221282 '..".' - rotr       $v0, $v0, 10
    0x0000F364: 0x7C412800 '.(A|' - ext        $at, $v0, 0, 6
    0x0000F368: 0x48E1001C '...H' - mtv        $at, S700
    0x0000F36C: 0xF3868084 '....' - vmzero.q   M100
    0x0000F370: 0x7C027804 '.x.|' - ins        $v0, $zr, 0, 16
    0x0000F374: 0xD2801C1C '....' - vi2f.s     S700, S700, 0
    0x0000F378: 0x3C0A0001 '...<' - lui        $t2, 0x1
    0x0000F37C: 0x14A002CE '....' - bnez       $a1, loc_0000FEB8
    0x0000F380: 0x35433400 '.4C5' - ori        $v1, $t2, 0x3400
    0x0000F384: 0x44820800 '...D' - mtc1       $v0, $fcr1
    0x0000F388: 0x3C010001 '...<' - lui        $at, 0x1
    0x0000F38C: 0xC4253404 '.4%.' - lwc1       $fpr05, 13316($at)
; Data ref 0x00119B70 ... 0x00000000 0x00000000 0x00000000 0x00000000 
    0x0000F390: 0x8E239B70 'p.#.' - lw         $v1, -25744($s1)
    0x0000F394: 0x468009A0 '...F' - cvt.s.w    $fpr06, $fpr01
    0x0000F398: 0x006E2824 '$(n.' - and        $a1, $v1, $t6
    0x0000F39C: 0x46053002 '.0.F' - mul.s      $fpr00, $fpr06, $fpr05
    0x0000F3A0: 0x14A00002 '....' - bnez       $a1, loc_0000F3AC
    0x0000F3A4: 0x46040040 '@..F' - add.s      $fpr01, $fpr00, $fpr04
    0x0000F3A8: 0x46030040 '@..F' - add.s      $fpr01, $fpr00, $fpr03

loc_0000F3AC:       ; Refs: 0x0000F3A0 
    0x0000F3AC: 0x3C020001 '...<' - lui        $v0, 0x1
    0x0000F3B0: 0x34433404 '.4C4' - ori        $v1, $v0, 0x3404

loc_0000F3B4:       ; Refs: 0x0000FEC8 0x0000FED4 
    0x0000F3B4: 0x44020800 '...D' - mfc1       $v0, $fcr1
    0x0000F3B8: 0x48E20000 '...H' - mtv        $v0, S000
    0x0000F3BC: 0xF3868088 '....' - vmzero.q   M200
    0x0000F3C0: 0x240BFE00 '...$' - li         $t3, -512
    0x0000F3C4: 0xF386808C '....' - vmzero.q   M300

loc_0000F3C8:       ; Refs: 0x0000F40C 0x0000F414 0x0000F41C ...
    0x0000F3C8: 0x953C0000 '..<.' - lhu        $gp, 0($t1)
    0x0000F3CC: 0x25290002 '..)%' - addiu      $t1, $t1, 2
    0x0000F3D0: 0x256B0008 '..k%' - addiu      $t3, $t3, 8
    0x0000F3D4: 0x003CE282 '..<.' - rotr       $gp, $gp, 10
    0x0000F3D8: 0x48FC003C '<..H' - mtv        $gp, S701
    0x0000F3DC: 0xD2803C3C '<<..' - vi2f.s     S701, S701, 0
    0x0000F3E0: 0x339C003F '?..3' - andi       $gp, $gp, 0x3F
    0x0000F3E4: 0x001CE0C0 '....' - sll        $gp, $gp, 3
    0x0000F3E8: 0x017C5821 '!X|.' - addu       $t3, $t3, $gp
    0x0000F3EC: 0x0160582D '-X`.' - min        $t3, $t3, $zr
    0x0000F3F0: 0x641C3C3C '<<.d' - vmul.s     S701, S701, S700
    0x0000F3F4: 0x0163E021 '!.c.' - addu       $gp, $t3, $v1
    0x0000F3F8: 0x3C010001 '...<' - lui        $at, 0x1
; Data ref 0x0000F604 ... 0xD043A0A0 0xD043A1A1 0xD043A2A2 0xD043A3A3 
    0x0000F3FC: 0x2421F604 '..!$' - addiu      $at, $at, -2556
    0x0000F400: 0x002B0821 '!.+.' - addu       $at, $at, $t3
    0x0000F404: 0x00200008 '.. .' - jr         $at
    0x0000F408: 0xCB9C0202 '....' - lv.s       S702, 512($gp)
    0x0000F40C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F410: 0x643C5C20 ' \<d' - vmul.s     S001, S702, S701
    0x0000F414: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F418: 0x643C5C08 '.\<d' - vmul.s     S200, S702, S701
    0x0000F41C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F420: 0x643C5C01 '.\<d' - vmul.s     S010, S702, S701
    0x0000F424: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F428: 0x643C5C28 '(\<d' - vmul.s     S201, S702, S701
    0x0000F42C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F430: 0x643C5C40 '@\<d' - vmul.s     S002, S702, S701
    0x0000F434: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F438: 0x643C5C60 '`\<d' - vmul.s     S003, S702, S701
    0x0000F43C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F440: 0x643C5C48 'H\<d' - vmul.s     S202, S702, S701
    0x0000F444: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F448: 0x643C5C21 '!\<d' - vmul.s     S011, S702, S701
    0x0000F44C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F450: 0x643C5C09 '.\<d' - vmul.s     S210, S702, S701
    0x0000F454: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F458: 0x643C5C02 '.\<d' - vmul.s     S020, S702, S701
    0x0000F45C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F460: 0x643C5C29 ')\<d' - vmul.s     S211, S702, S701
    0x0000F464: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F468: 0x643C5C41 'A\<d' - vmul.s     S012, S702, S701
    0x0000F46C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F470: 0x643C5C68 'h\<d' - vmul.s     S203, S702, S701
    0x0000F474: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F478: 0x643C5C04 '.\<d' - vmul.s     S100, S702, S701
    0x0000F47C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F480: 0x643C5C24 '$\<d' - vmul.s     S101, S702, S701
    0x0000F484: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F488: 0x643C5C0C '.\<d' - vmul.s     S300, S702, S701
    0x0000F48C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F490: 0x643C5C61 'a\<d' - vmul.s     S013, S702, S701
    0x0000F494: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F498: 0x643C5C49 'I\<d' - vmul.s     S212, S702, S701
    0x0000F49C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F4A0: 0x643C5C22 '"\<d' - vmul.s     S021, S702, S701
    0x0000F4A4: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F4A8: 0x643C5C0A '.\<d' - vmul.s     S220, S702, S701
    0x0000F4AC: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F4B0: 0x643C5C03 '.\<d' - vmul.s     S030, S702, S701
    0x0000F4B4: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F4B8: 0x643C5C2A '*\<d' - vmul.s     S221, S702, S701
    0x0000F4BC: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F4C0: 0x643C5C42 'B\<d' - vmul.s     S022, S702, S701
    0x0000F4C4: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F4C8: 0x643C5C69 'i\<d' - vmul.s     S213, S702, S701
    0x0000F4CC: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F4D0: 0x643C5C05 '.\<d' - vmul.s     S110, S702, S701
    0x0000F4D4: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F4D8: 0x643C5C2C ',\<d' - vmul.s     S301, S702, S701
    0x0000F4DC: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F4E0: 0x643C5C44 'D\<d' - vmul.s     S102, S702, S701
    0x0000F4E4: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F4E8: 0x643C5C64 'd\<d' - vmul.s     S103, S702, S701
    0x0000F4EC: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F4F0: 0x643C5C4C 'L\<d' - vmul.s     S302, S702, S701
    0x0000F4F4: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F4F8: 0x643C5C25 '%\<d' - vmul.s     S111, S702, S701
    0x0000F4FC: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F500: 0x643C5C0D '.\<d' - vmul.s     S310, S702, S701
    0x0000F504: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F508: 0x643C5C62 'b\<d' - vmul.s     S023, S702, S701
    0x0000F50C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F510: 0x643C5C4A 'J\<d' - vmul.s     S222, S702, S701
    0x0000F514: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F518: 0x643C5C23 '#\<d' - vmul.s     S031, S702, S701
    0x0000F51C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F520: 0x643C5C0B '.\<d' - vmul.s     S230, S702, S701
    0x0000F524: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F528: 0x643C5C2B '+\<d' - vmul.s     S231, S702, S701
    0x0000F52C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F530: 0x643C5C43 'C\<d' - vmul.s     S032, S702, S701
    0x0000F534: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F538: 0x643C5C6A 'j\<d' - vmul.s     S223, S702, S701
    0x0000F53C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F540: 0x643C5C06 '.\<d' - vmul.s     S120, S702, S701
    0x0000F544: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F548: 0x643C5C2D '-\<d' - vmul.s     S311, S702, S701
    0x0000F54C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F550: 0x643C5C45 'E\<d' - vmul.s     S112, S702, S701
    0x0000F554: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F558: 0x643C5C6C 'l\<d' - vmul.s     S303, S702, S701
    0x0000F55C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F560: 0x643C5C65 'e\<d' - vmul.s     S113, S702, S701
    0x0000F564: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F568: 0x643C5C4D 'M\<d' - vmul.s     S312, S702, S701
    0x0000F56C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F570: 0x643C5C26 '&\<d' - vmul.s     S121, S702, S701
    0x0000F574: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F578: 0x643C5C0E '.\<d' - vmul.s     S320, S702, S701
    0x0000F57C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F580: 0x643C5C63 'c\<d' - vmul.s     S033, S702, S701
    0x0000F584: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F588: 0x643C5C4B 'K\<d' - vmul.s     S232, S702, S701
    0x0000F58C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F590: 0x643C5C6B 'k\<d' - vmul.s     S233, S702, S701
    0x0000F594: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F598: 0x643C5C07 '.\<d' - vmul.s     S130, S702, S701
    0x0000F59C: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F5A0: 0x643C5C2E '.\<d' - vmul.s     S321, S702, S701
    0x0000F5A4: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F5A8: 0x643C5C46 'F\<d' - vmul.s     S122, S702, S701
    0x0000F5AC: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F5B0: 0x643C5C6D 'm\<d' - vmul.s     S313, S702, S701
    0x0000F5B4: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F5B8: 0x643C5C66 'f\<d' - vmul.s     S123, S702, S701
    0x0000F5BC: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F5C0: 0x643C5C4E 'N\<d' - vmul.s     S322, S702, S701
    0x0000F5C4: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F5C8: 0x643C5C27 ''\<d' - vmul.s     S131, S702, S701
    0x0000F5CC: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F5D0: 0x643C5C0F '.\<d' - vmul.s     S330, S702, S701
    0x0000F5D4: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F5D8: 0x643C5C2F '/\<d' - vmul.s     S331, S702, S701
    0x0000F5DC: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F5E0: 0x643C5C47 'G\<d' - vmul.s     S132, S702, S701
    0x0000F5E4: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F5E8: 0x643C5C6E 'n\<d' - vmul.s     S323, S702, S701
    0x0000F5EC: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F5F0: 0x643C5C67 'g\<d' - vmul.s     S133, S702, S701
    0x0000F5F4: 0x08003CF2 '.<..' - j          loc_0000F3C8
    0x0000F5F8: 0x643C5C4F 'O\<d' - vmul.s     S332, S702, S701
    0x0000F5FC: 0x00000000 '....' - nop        
    0x0000F600: 0x643C5C6F 'o\<d' - vmul.s     S333, S702, S701
    0x0000F604: 0xD043A0A0 '..C.' - vbfy2.q    R000, R000
    0x0000F608: 0xD043A1A1 '..C.' - vbfy2.q    R001, R001
    0x0000F60C: 0xD043A2A2 '..C.' - vbfy2.q    R002, R002
    0x0000F610: 0xD043A3A3 '..C.' - vbfy2.q    R003, R003
    0x0000F614: 0xD043A4A4 '..C.' - vbfy2.q    R100, R100
    0x0000F618: 0xD043A5A5 '..C.' - vbfy2.q    R101, R101
    0x0000F61C: 0xD043A6A6 '..C.' - vbfy2.q    R102, R102
    0x0000F620: 0xD043A7A7 '..C.' - vbfy2.q    R103, R103
    0x0000F624: 0x65718383 '..qe' - vscl.q     C030, C030, S413
    0x0000F628: 0x65718787 '..qe' - vscl.q     C130, C130, S413
    0x0000F62C: 0x60818383 '...`' - vsub.q     C030, C030, C010
    0x0000F630: 0x60858787 '...`' - vsub.q     C130, C130, C110
    0x0000F634: 0xD042A0A0 '..B.' - vbfy1.q    R000, R000
    0x0000F638: 0xD042A1A1 '..B.' - vbfy1.q    R001, R001
    0x0000F63C: 0xD042A2A2 '..B.' - vbfy1.q    R002, R002
    0x0000F640: 0xD042A3A3 '..B.' - vbfy1.q    R003, R003
    0x0000F644: 0xD042A4A4 '..B.' - vbfy1.q    R100, R100
    0x0000F648: 0xD042A5A5 '..B.' - vbfy1.q    R101, R101
    0x0000F64C: 0xD042A6A6 '..B.' - vbfy1.q    R102, R102
    0x0000F650: 0xD042A7A7 '..B.' - vbfy1.q    R103, R103
    0x0000F654: 0x60098A94 '...`' - vadd.q     C500, C220, C210
    0x0000F658: 0x600D8E98 '...`' - vadd.q     C600, C320, C310
    0x0000F65C: 0x60898A95 '...`' - vsub.q     C510, C220, C210
    0x0000F660: 0x608D8E99 '...`' - vsub.q     C610, C320, C310
    0x0000F664: 0x600B8896 '...`' - vadd.q     C520, C200, C230
    0x0000F668: 0x600F8C9A '...`' - vadd.q     C620, C300, C330
    0x0000F66C: 0x608B8897 '...`' - vsub.q     C530, C200, C230
    0x0000F670: 0x608F8C9B '...`' - vsub.q     C630, C300, C330
    0x0000F674: 0x6014968B '...`' - vadd.q     C230, C520, C500
    0x0000F678: 0x60189A8C '...`' - vadd.q     C300, C620, C600
    0x0000F67C: 0x6094968A '...`' - vsub.q     C220, C520, C500
    0x0000F680: 0x60989A8D '...`' - vsub.q     C310, C620, C600
    0x0000F684: 0x6017959D '...`' - vadd.q     C710, C510, C530
    0x0000F688: 0x601B999E '...`' - vadd.q     C720, C610, C630
    0x0000F68C: 0x65718A8A '..qe' - vscl.q     C220, C220, S413
    0x0000F690: 0x65718D8D '..qe' - vscl.q     C310, C310, S413
    0x0000F694: 0x65729D9D '..re' - vscl.q     C710, C710, S423
    0x0000F698: 0x65729E9E '..re' - vscl.q     C720, C720, S423
    0x0000F69C: 0x65709789 '..pe' - vscl.q     C210, C530, S403
    0x0000F6A0: 0x65709B8E '..pe' - vscl.q     C320, C630, S403
    0x0000F6A4: 0x609D8989 '...`' - vsub.q     C210, C210, C710
    0x0000F6A8: 0x609E8E8E '...`' - vsub.q     C320, C320, C720
    0x0000F6AC: 0x65739588 '..se' - vscl.q     C200, C510, S433
    0x0000F6B0: 0x6573998F '..se' - vscl.q     C330, C610, S433
    0x0000F6B4: 0x60889D88 '...`' - vsub.q     C200, C710, C200
    0x0000F6B8: 0x608F9E8F '...`' - vsub.q     C330, C720, C330
    0x0000F6BC: 0x608B8888 '...`' - vsub.q     C200, C200, C230
    0x0000F6C0: 0x608C8F8F '...`' - vsub.q     C330, C330, C300
    0x0000F6C4: 0x60888A8A '...`' - vsub.q     C220, C220, C200
    0x0000F6C8: 0x608F8D8D '...`' - vsub.q     C310, C310, C330
    0x0000F6CC: 0x600A8989 '...`' - vadd.q     C210, C210, C220
    0x0000F6D0: 0x600D8E8E '...`' - vadd.q     C320, C320, C310
    0x0000F6D4: 0x600B8094 '...`' - vadd.q     C500, C000, C230
    0x0000F6D8: 0x60888297 '...`' - vsub.q     C530, C020, C200
    0x0000F6DC: 0x600A8395 '...`' - vadd.q     C510, C030, C220
    0x0000F6E0: 0x60098196 '...`' - vadd.q     C520, C010, C210
    0x0000F6E4: 0x608C849F '...`' - vsub.q     C730, C100, C300
    0x0000F6E8: 0x600F869C '...`' - vadd.q     C700, C120, C330
    0x0000F6EC: 0x608D879E '...`' - vsub.q     C720, C130, C310
    0x0000F6F0: 0x608E859D '...`' - vsub.q     C710, C110, C320
    0x0000F6F4: 0x600C848C '...`' - vadd.q     C300, C100, C300
    0x0000F6F8: 0x608F868F '...`' - vsub.q     C330, C120, C330
    0x0000F6FC: 0x600D878D '...`' - vadd.q     C310, C130, C310
    0x0000F700: 0x600E858E '...`' - vadd.q     C320, C110, C320
    0x0000F704: 0x608B809B '...`' - vsub.q     C630, C000, C230
    0x0000F708: 0x60088298 '...`' - vadd.q     C600, C020, C200
    0x0000F70C: 0x608A839A '...`' - vsub.q     C620, C030, C220
    0x0000F710: 0x60898199 '...`' - vsub.q     C610, C010, C210
    0x0000F714: 0x602CB4A0 '..,`' - vadd.q     R000, R500, R300
    0x0000F718: 0x60ACB4A1 '...`' - vsub.q     R001, R500, R300
    0x0000F71C: 0x602EB6A2 '...`' - vadd.q     R002, R502, R302
    0x0000F720: 0x60AEB6A3 '...`' - vsub.q     R003, R502, R302
    0x0000F724: 0x60B7ADA6 '...`' - vsub.q     R102, R301, R503
    0x0000F728: 0x602FB5A5 '../`' - vadd.q     R101, R501, R303
    0x0000F72C: 0x60AFB5A7 '...`' - vsub.q     R103, R501, R303
    0x0000F730: 0x6037ADA4 '..7`' - vadd.q     R100, R301, R503
    0x0000F734: 0x603CB8A8 '..<`' - vadd.q     R200, R600, R700
    0x0000F738: 0x60BCB8A9 '...`' - vsub.q     R201, R600, R700
    0x0000F73C: 0x603EBAAA '..>`' - vadd.q     R202, R602, R702
    0x0000F740: 0x60BEBAAB '...`' - vsub.q     R203, R602, R702
    0x0000F744: 0x603BBDAC '..;`' - vadd.q     R300, R701, R603
    0x0000F748: 0x60BBBDAD '...`' - vsub.q     R301, R701, R603
    0x0000F74C: 0x603FB9AE '..?`' - vadd.q     R302, R601, R703
    0x0000F750: 0x60BFB9AF '...`' - vsub.q     R303, R601, R703
    0x0000F754: 0x6571A3A3 '..qe' - vscl.q     R003, R003, S413
    0x0000F758: 0x6571ABAB '..qe' - vscl.q     R203, R203, S413
    0x0000F75C: 0x60A2A3A3 '...`' - vsub.q     R003, R003, R002
    0x0000F760: 0x60AAABAB '...`' - vsub.q     R203, R203, R202
    0x0000F764: 0xD0438080 '..C.' - vbfy2.q    C000, C000
    0x0000F768: 0xD0438888 '..C.' - vbfy2.q    C200, C200
    0x0000F76C: 0xD0438181 '..C.' - vbfy2.q    C010, C010
    0x0000F770: 0xD0438989 '..C.' - vbfy2.q    C210, C210
    0x0000F774: 0xD0438282 '..C.' - vbfy2.q    C020, C020
    0x0000F778: 0xD0438A8A '..C.' - vbfy2.q    C220, C220
    0x0000F77C: 0xD0438383 '..C.' - vbfy2.q    C030, C030
    0x0000F780: 0xD0438B8B '..C.' - vbfy2.q    C230, C230
    0x0000F784: 0x6024A5B6 '..$`' - vadd.q     R502, R101, R100
    0x0000F788: 0x602CAEBA '..,`' - vadd.q     R602, R302, R300
    0x0000F78C: 0x60A4A5A5 '...`' - vsub.q     R101, R101, R100
    0x0000F790: 0x60ACAEAE '...`' - vsub.q     R302, R302, R300
    0x0000F794: 0x6571A5A5 '..qe' - vscl.q     R101, R101, S413
    0x0000F798: 0x6571AEAE '..qe' - vscl.q     R302, R302, S413
    0x0000F79C: 0x6027A6A4 '..'`' - vadd.q     R100, R102, R103
    0x0000F7A0: 0x602FADAC '../`' - vadd.q     R300, R301, R303
    0x0000F7A4: 0x6572A4A4 '..re' - vscl.q     R100, R100, S423
    0x0000F7A8: 0x6572ACAC '..re' - vscl.q     R300, R300, S423
    0x0000F7AC: 0x6570A7A7 '..pe' - vscl.q     R103, R103, S403
    0x0000F7B0: 0x6570AFAF '..pe' - vscl.q     R303, R303, S403
    0x0000F7B4: 0x60A4A7A7 '...`' - vsub.q     R103, R103, R100
    0x0000F7B8: 0x60ACAFAF '...`' - vsub.q     R303, R303, R300
    0x0000F7BC: 0x6573A6A6 '..se' - vscl.q     R102, R102, S433
    0x0000F7C0: 0x6573ADAD '..se' - vscl.q     R301, R301, S433
    0x0000F7C4: 0x60A6A4A6 '...`' - vsub.q     R102, R100, R102
    0x0000F7C8: 0x60ADACAD '...`' - vsub.q     R301, R300, R301
    0x0000F7CC: 0x60B6A6A6 '...`' - vsub.q     R102, R102, R502
    0x0000F7D0: 0x60BAADAD '...`' - vsub.q     R301, R301, R602
    0x0000F7D4: 0x60A6A5A5 '...`' - vsub.q     R101, R101, R102
    0x0000F7D8: 0x60ADAEAE '...`' - vsub.q     R302, R302, R301
    0x0000F7DC: 0x6025A7A4 '..%`' - vadd.q     R100, R103, R101
    0x0000F7E0: 0x602EAFAC '...`' - vadd.q     R300, R303, R302
    0x0000F7E4: 0x6036A0B4 '..6`' - vadd.q     R500, R000, R502
    0x0000F7E8: 0x603AA8B8 '..:`' - vadd.q     R600, R200, R602
    0x0000F7EC: 0x60BAA8BF '...`' - vsub.q     R703, R200, R602
    0x0000F7F0: 0x60B6A0A7 '...`' - vsub.q     R103, R000, R502
    0x0000F7F4: 0x6026A1B5 '..&`' - vadd.q     R501, R001, R102
    0x0000F7F8: 0x602DA9B9 '..-`' - vadd.q     R601, R201, R301
    0x0000F7FC: 0x60ADA9BE '...`' - vsub.q     R702, R201, R301
    0x0000F800: 0x60A6A1A6 '...`' - vsub.q     R102, R001, R102
    0x0000F804: 0x6025A3B6 '..%`' - vadd.q     R502, R003, R101
    0x0000F808: 0x602EABBA '...`' - vadd.q     R602, R203, R302
    0x0000F80C: 0x60AEABBD '...`' - vsub.q     R701, R203, R302
    0x0000F810: 0x60A5A3A5 '...`' - vsub.q     R101, R003, R101
    0x0000F814: 0x60A4A2B7 '...`' - vsub.q     R503, R002, R100
    0x0000F818: 0x60ACAABB '...`' - vsub.q     R603, R202, R300
    0x0000F81C: 0x602CAABC '..,`' - vadd.q     R700, R202, R300
    0x0000F820: 0x6024A2A4 '..$`' - vadd.q     R100, R002, R100
    0x0000F824: 0x10800192 '....' - beqz       $a0, loc_0000FE70
    0x0000F828: 0x32650001 '..e2' - andi       $a1, $s3, 0x1
    0x0000F82C: 0x326B0002 '..k2' - andi       $t3, $s3, 0x2
    0x0000F830: 0x000B1080 '....' - sll        $v0, $t3, 2
    0x0000F834: 0x00455021 '!PE.' - addu       $t2, $v0, $a1
    0x0000F838: 0x000B20C0 '. ..' - sll        $a0, $t3, 3
    0x0000F83C: 0x000AA100 '....' - sll        $s4, $t2, 4
    0x0000F840: 0x00855821 '!X..' - addu       $t3, $a0, $a1
    0x0000F844: 0x028FA821 '!...' - addu       $s5, $s4, $t7
    0x0000F848: 0x000BA0C0 '....' - sll        $s4, $t3, 3
    0x0000F84C: 0xDAAB0100 '....' - lv.q       C230, 256($s5)
    0x0000F850: 0xDAA30000 '....' - lv.q       C030, 0($s5)
    0x0000F854: 0x65108B80 '...e' - vscl.q     C000, C230, S400
    0x0000F858: 0x65118381 '...e' - vscl.q     C010, C030, S410
    0x0000F85C: 0x60018080 '...`' - vadd.q     C000, C000, C010
    0x0000F860: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000F864: 0x60039488 '...`' - vadd.q     C200, C500, C030
    0x0000F868: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000F86C: 0x600B948A '...`' - vadd.q     C220, C500, C230
    0x0000F870: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000F874: 0x60809489 '...`' - vsub.q     C210, C500, C000
    0x0000F878: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000F87C: 0x6003848C '...`' - vadd.q     C300, C100, C030
    0x0000F880: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000F884: 0x600B848E '...`' - vadd.q     C320, C100, C230
    0x0000F888: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000F88C: 0x6080848D '...`' - vsub.q     C310, C100, C000
    0x0000F890: 0xD2178888 '....' - vf2in.q    C200, C200, 23
    0x0000F894: 0xD2178A8A '....' - vf2in.q    C220, C220, 23
    0x0000F898: 0xD2178989 '....' - vf2in.q    C210, C210, 23
    0x0000F89C: 0xD2178C8C '....' - vf2in.q    C300, C300, 23
    0x0000F8A0: 0xD2178E8E '....' - vf2in.q    C320, C320, 23
    0x0000F8A4: 0xD2178D8D '....' - vf2in.q    C310, C310, 23
    0x0000F8A8: 0xD03CA888 '..<.' - vi2uc.q    C200, R200
    0x0000F8AC: 0xD03CA9A8 '..<.' - vi2uc.q    R200, R201
    0x0000F8B0: 0xD03CAAC8 '..<.' - vi2uc.q    C200, R202
    0x0000F8B4: 0xD03CABE8 '..<.' - vi2uc.q    R200, R203
    0x0000F8B8: 0xD03CAC8C '..<.' - vi2uc.q    C300, R300
    0x0000F8BC: 0xD03CADAC '..<.' - vi2uc.q    R300, R301
    0x0000F8C0: 0xD03CAECC '..<.' - vi2uc.q    C300, R302
    0x0000F8C4: 0xD03CAFEC '..<.' - vi2uc.q    R300, R303
    0x0000F8C8: 0xFA480000 '..H.' - sv.q       C200, 0($s2)
    0x0000F8CC: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000F8D0: 0x60039888 '...`' - vadd.q     C200, C600, C030
    0x0000F8D4: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000F8D8: 0x600B988A '...`' - vadd.q     C220, C600, C230
    0x0000F8DC: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000F8E0: 0x60809889 '...`' - vsub.q     C210, C600, C000
    0x0000F8E4: 0xFA4C0010 '..L.' - sv.q       C300, 16($s2)
    0x0000F8E8: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000F8EC: 0x60039C8C '...`' - vadd.q     C300, C700, C030
    0x0000F8F0: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000F8F4: 0x600B9C8E '...`' - vadd.q     C320, C700, C230
    0x0000F8F8: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000F8FC: 0x60809C8D '...`' - vsub.q     C310, C700, C000
    0x0000F900: 0xD2178888 '....' - vf2in.q    C200, C200, 23
    0x0000F904: 0xD2178A8A '....' - vf2in.q    C220, C220, 23
    0x0000F908: 0xD2178989 '....' - vf2in.q    C210, C210, 23
    0x0000F90C: 0xD2178C8C '....' - vf2in.q    C300, C300, 23
    0x0000F910: 0xD2178E8E '....' - vf2in.q    C320, C320, 23
    0x0000F914: 0xD2178D8D '....' - vf2in.q    C310, C310, 23
    0x0000F918: 0xD03CA888 '..<.' - vi2uc.q    C200, R200
    0x0000F91C: 0xD03CA9A8 '..<.' - vi2uc.q    R200, R201
    0x0000F920: 0xD03CAAC8 '..<.' - vi2uc.q    C200, R202
    0x0000F924: 0xD03CABE8 '..<.' - vi2uc.q    R200, R203
    0x0000F928: 0xD03CAC8C '..<.' - vi2uc.q    C300, R300
    0x0000F92C: 0xD03CADAC '..<.' - vi2uc.q    R300, R301
    0x0000F930: 0xD03CAECC '..<.' - vi2uc.q    C300, R302
    0x0000F934: 0xD03CAFEC '..<.' - vi2uc.q    R300, R303
    0x0000F938: 0xDAAB0120 ' ...' - lv.q       C230, 288($s5)
    0x0000F93C: 0xDAA30020 ' ...' - lv.q       C030, 32($s5)
    0x0000F940: 0xFA480020 ' .H.' - sv.q       C200, 32($s2)
    0x0000F944: 0xFA4C0030 '0.L.' - sv.q       C300, 48($s2)
    0x0000F948: 0x65108B80 '...e' - vscl.q     C000, C230, S400
    0x0000F94C: 0x65118381 '...e' - vscl.q     C010, C030, S410
    0x0000F950: 0x60018080 '...`' - vadd.q     C000, C000, C010
    0x0000F954: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000F958: 0x60039588 '...`' - vadd.q     C200, C510, C030
    0x0000F95C: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000F960: 0x600B958A '...`' - vadd.q     C220, C510, C230
    0x0000F964: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000F968: 0x60809589 '...`' - vsub.q     C210, C510, C000
    0x0000F96C: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000F970: 0x6003858C '...`' - vadd.q     C300, C110, C030
    0x0000F974: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000F978: 0x600B858E '...`' - vadd.q     C320, C110, C230
    0x0000F97C: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000F980: 0x6080858D '...`' - vsub.q     C310, C110, C000
    0x0000F984: 0xD2178888 '....' - vf2in.q    C200, C200, 23
    0x0000F988: 0xD2178A8A '....' - vf2in.q    C220, C220, 23
    0x0000F98C: 0xD2178989 '....' - vf2in.q    C210, C210, 23
    0x0000F990: 0xD2178C8C '....' - vf2in.q    C300, C300, 23
    0x0000F994: 0xD2178E8E '....' - vf2in.q    C320, C320, 23
    0x0000F998: 0xD2178D8D '....' - vf2in.q    C310, C310, 23
    0x0000F99C: 0xD03CA888 '..<.' - vi2uc.q    C200, R200
    0x0000F9A0: 0xD03CA9A8 '..<.' - vi2uc.q    R200, R201
    0x0000F9A4: 0xD03CAAC8 '..<.' - vi2uc.q    C200, R202
    0x0000F9A8: 0xD03CABE8 '..<.' - vi2uc.q    R200, R203
    0x0000F9AC: 0xD03CAC8C '..<.' - vi2uc.q    C300, R300
    0x0000F9B0: 0xD03CADAC '..<.' - vi2uc.q    R300, R301
    0x0000F9B4: 0xD03CAECC '..<.' - vi2uc.q    C300, R302
    0x0000F9B8: 0xD03CAFEC '..<.' - vi2uc.q    R300, R303
    0x0000F9BC: 0xFA480040 '@.H.' - sv.q       C200, 64($s2)
    0x0000F9C0: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000F9C4: 0x60039988 '...`' - vadd.q     C200, C610, C030
    0x0000F9C8: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000F9CC: 0x600B998A '...`' - vadd.q     C220, C610, C230
    0x0000F9D0: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000F9D4: 0x60809989 '...`' - vsub.q     C210, C610, C000
    0x0000F9D8: 0xFA4C0050 'P.L.' - sv.q       C300, 80($s2)
    0x0000F9DC: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000F9E0: 0x60039D8C '...`' - vadd.q     C300, C710, C030
    0x0000F9E4: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000F9E8: 0x600B9D8E '...`' - vadd.q     C320, C710, C230
    0x0000F9EC: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000F9F0: 0x60809D8D '...`' - vsub.q     C310, C710, C000
    0x0000F9F4: 0xD2178888 '....' - vf2in.q    C200, C200, 23
    0x0000F9F8: 0xD2178A8A '....' - vf2in.q    C220, C220, 23
    0x0000F9FC: 0xD2178989 '....' - vf2in.q    C210, C210, 23
    0x0000FA00: 0xD2178C8C '....' - vf2in.q    C300, C300, 23
    0x0000FA04: 0xD2178E8E '....' - vf2in.q    C320, C320, 23
    0x0000FA08: 0xD2178D8D '....' - vf2in.q    C310, C310, 23
    0x0000FA0C: 0xD03CA888 '..<.' - vi2uc.q    C200, R200
    0x0000FA10: 0xD03CA9A8 '..<.' - vi2uc.q    R200, R201
    0x0000FA14: 0xD03CAAC8 '..<.' - vi2uc.q    C200, R202
    0x0000FA18: 0xD03CABE8 '..<.' - vi2uc.q    R200, R203
    0x0000FA1C: 0xD03CAC8C '..<.' - vi2uc.q    C300, R300
    0x0000FA20: 0xD03CADAC '..<.' - vi2uc.q    R300, R301
    0x0000FA24: 0xD03CAECC '..<.' - vi2uc.q    C300, R302
    0x0000FA28: 0xD03CAFEC '..<.' - vi2uc.q    R300, R303
    0x0000FA2C: 0xDAAB0140 '@...' - lv.q       C230, 320($s5)
    0x0000FA30: 0xDAA30040 '@...' - lv.q       C030, 64($s5)
    0x0000FA34: 0xFA480060 '`.H.' - sv.q       C200, 96($s2)
    0x0000FA38: 0xFA4C0070 'p.L.' - sv.q       C300, 112($s2)
    0x0000FA3C: 0x65108B80 '...e' - vscl.q     C000, C230, S400
    0x0000FA40: 0x65118381 '...e' - vscl.q     C010, C030, S410
    0x0000FA44: 0x60018080 '...`' - vadd.q     C000, C000, C010
    0x0000FA48: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000FA4C: 0x60039688 '...`' - vadd.q     C200, C520, C030
    0x0000FA50: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000FA54: 0x600B968A '...`' - vadd.q     C220, C520, C230
    0x0000FA58: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000FA5C: 0x60809689 '...`' - vsub.q     C210, C520, C000
    0x0000FA60: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000FA64: 0x6003868C '...`' - vadd.q     C300, C120, C030
    0x0000FA68: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000FA6C: 0x600B868E '...`' - vadd.q     C320, C120, C230
    0x0000FA70: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000FA74: 0x6080868D '...`' - vsub.q     C310, C120, C000
    0x0000FA78: 0xD2178888 '....' - vf2in.q    C200, C200, 23
    0x0000FA7C: 0xD2178A8A '....' - vf2in.q    C220, C220, 23
    0x0000FA80: 0xD2178989 '....' - vf2in.q    C210, C210, 23
    0x0000FA84: 0xD2178C8C '....' - vf2in.q    C300, C300, 23
    0x0000FA88: 0xD2178E8E '....' - vf2in.q    C320, C320, 23
    0x0000FA8C: 0xD2178D8D '....' - vf2in.q    C310, C310, 23
    0x0000FA90: 0xD03CA888 '..<.' - vi2uc.q    C200, R200
    0x0000FA94: 0xD03CA9A8 '..<.' - vi2uc.q    R200, R201
    0x0000FA98: 0xD03CAAC8 '..<.' - vi2uc.q    C200, R202
    0x0000FA9C: 0xD03CABE8 '..<.' - vi2uc.q    R200, R203
    0x0000FAA0: 0xD03CAC8C '..<.' - vi2uc.q    C300, R300
    0x0000FAA4: 0xD03CADAC '..<.' - vi2uc.q    R300, R301
    0x0000FAA8: 0xD03CAECC '..<.' - vi2uc.q    C300, R302
    0x0000FAAC: 0xD03CAFEC '..<.' - vi2uc.q    R300, R303
    0x0000FAB0: 0xFA480080 '..H.' - sv.q       C200, 128($s2)
    0x0000FAB4: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000FAB8: 0x60039A88 '...`' - vadd.q     C200, C620, C030
    0x0000FABC: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000FAC0: 0x600B9A8A '...`' - vadd.q     C220, C620, C230
    0x0000FAC4: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000FAC8: 0x60809A89 '...`' - vsub.q     C210, C620, C000
    0x0000FACC: 0xFA4C0090 '..L.' - sv.q       C300, 144($s2)
    0x0000FAD0: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000FAD4: 0x60039E8C '...`' - vadd.q     C300, C720, C030
    0x0000FAD8: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000FADC: 0x600B9E8E '...`' - vadd.q     C320, C720, C230
    0x0000FAE0: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000FAE4: 0x60809E8D '...`' - vsub.q     C310, C720, C000
    0x0000FAE8: 0xD2178888 '....' - vf2in.q    C200, C200, 23
    0x0000FAEC: 0xD2178A8A '....' - vf2in.q    C220, C220, 23
    0x0000FAF0: 0xD2178989 '....' - vf2in.q    C210, C210, 23
    0x0000FAF4: 0xD2178C8C '....' - vf2in.q    C300, C300, 23
    0x0000FAF8: 0xD2178E8E '....' - vf2in.q    C320, C320, 23
    0x0000FAFC: 0xD2178D8D '....' - vf2in.q    C310, C310, 23
    0x0000FB00: 0xD03CA888 '..<.' - vi2uc.q    C200, R200
    0x0000FB04: 0xD03CA9A8 '..<.' - vi2uc.q    R200, R201
    0x0000FB08: 0xD03CAAC8 '..<.' - vi2uc.q    C200, R202
    0x0000FB0C: 0xD03CABE8 '..<.' - vi2uc.q    R200, R203
    0x0000FB10: 0xD03CAC8C '..<.' - vi2uc.q    C300, R300
    0x0000FB14: 0xD03CADAC '..<.' - vi2uc.q    R300, R301
    0x0000FB18: 0xD03CAECC '..<.' - vi2uc.q    C300, R302
    0x0000FB1C: 0xD03CAFEC '..<.' - vi2uc.q    R300, R303
    0x0000FB20: 0xDAAB0160 '`...' - lv.q       C230, 352($s5)
    0x0000FB24: 0xDAA30060 '`...' - lv.q       C030, 96($s5)
    0x0000FB28: 0xFA4800A0 '..H.' - sv.q       C200, 160($s2)
    0x0000FB2C: 0xFA4C00B0 '..L.' - sv.q       C300, 176($s2)
    0x0000FB30: 0x65108B80 '...e' - vscl.q     C000, C230, S400
    0x0000FB34: 0x65118381 '...e' - vscl.q     C010, C030, S410
    0x0000FB38: 0x60018080 '...`' - vadd.q     C000, C000, C010
    0x0000FB3C: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000FB40: 0x60039788 '...`' - vadd.q     C200, C530, C030
    0x0000FB44: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000FB48: 0x600B978A '...`' - vadd.q     C220, C530, C230
    0x0000FB4C: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000FB50: 0x60809789 '...`' - vsub.q     C210, C530, C000
    0x0000FB54: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000FB58: 0x6003878C '...`' - vadd.q     C300, C130, C030
    0x0000FB5C: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000FB60: 0x600B878E '...`' - vadd.q     C320, C130, C230
    0x0000FB64: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000FB68: 0x6080878D '...`' - vsub.q     C310, C130, C000
    0x0000FB6C: 0xD2178888 '....' - vf2in.q    C200, C200, 23
    0x0000FB70: 0xD2178A8A '....' - vf2in.q    C220, C220, 23
    0x0000FB74: 0xD2178989 '....' - vf2in.q    C210, C210, 23
    0x0000FB78: 0xD2178C8C '....' - vf2in.q    C300, C300, 23
    0x0000FB7C: 0xD2178E8E '....' - vf2in.q    C320, C320, 23
    0x0000FB80: 0xD2178D8D '....' - vf2in.q    C310, C310, 23
    0x0000FB84: 0xD03CA888 '..<.' - vi2uc.q    C200, R200
    0x0000FB88: 0xD03CA9A8 '..<.' - vi2uc.q    R200, R201
    0x0000FB8C: 0xD03CAAC8 '..<.' - vi2uc.q    C200, R202
    0x0000FB90: 0xD03CABE8 '..<.' - vi2uc.q    R200, R203
    0x0000FB94: 0xD03CAC8C '..<.' - vi2uc.q    C300, R300
    0x0000FB98: 0xD03CADAC '..<.' - vi2uc.q    R300, R301
    0x0000FB9C: 0xD03CAECC '..<.' - vi2uc.q    C300, R302
    0x0000FBA0: 0xD03CAFEC '..<.' - vi2uc.q    R300, R303
    0x0000FBA4: 0xFA4800C0 '..H.' - sv.q       C200, 192($s2)
    0x0000FBA8: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000FBAC: 0x60039B88 '...`' - vadd.q     C200, C630, C030
    0x0000FBB0: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000FBB4: 0x600B9B8A '...`' - vadd.q     C220, C630, C230
    0x0000FBB8: 0xDD000050 'P...' - vpfxt      [x, x, y, y]
    0x0000FBBC: 0x60809B89 '...`' - vsub.q     C210, C630, C000
    0x0000FBC0: 0xFA4C00D0 '..L.' - sv.q       C300, 208($s2)
    0x0000FBC4: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000FBC8: 0x60039F8C '...`' - vadd.q     C300, C730, C030
    0x0000FBCC: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000FBD0: 0x600B9F8E '...`' - vadd.q     C320, C730, C230
    0x0000FBD4: 0xDD0000FA '....' - vpfxt      [z, z, w, w]
    0x0000FBD8: 0x60809F8D '...`' - vsub.q     C310, C730, C000
    0x0000FBDC: 0xD2178888 '....' - vf2in.q    C200, C200, 23
    0x0000FBE0: 0xD2178A8A '....' - vf2in.q    C220, C220, 23
    0x0000FBE4: 0xD2178989 '....' - vf2in.q    C210, C210, 23
    0x0000FBE8: 0xD2178C8C '....' - vf2in.q    C300, C300, 23
    0x0000FBEC: 0xD2178E8E '....' - vf2in.q    C320, C320, 23
    0x0000FBF0: 0xD2178D8D '....' - vf2in.q    C310, C310, 23
    0x0000FBF4: 0xD03CA888 '..<.' - vi2uc.q    C200, R200
    0x0000FBF8: 0xD03CA9A8 '..<.' - vi2uc.q    R200, R201
    0x0000FBFC: 0xD03CAAC8 '..<.' - vi2uc.q    C200, R202
    0x0000FC00: 0xD03CABE8 '..<.' - vi2uc.q    R200, R203
    0x0000FC04: 0xD03CAC8C '..<.' - vi2uc.q    C300, R300
    0x0000FC08: 0xD03CADAC '..<.' - vi2uc.q    R300, R301
    0x0000FC0C: 0xD03CAECC '..<.' - vi2uc.q    C300, R302
    0x0000FC10: 0xD03CAFEC '..<.' - vi2uc.q    R300, R303
; Data ref 0x00119B70 ... 0x00000000 0x00000000 0x00000000 0x00000000 
    0x0000FC14: 0x8E3C9B70 'p.<.' - lw         $gp, -25744($s1)
    0x0000FC18: 0x038E2024 '$ ..' - and        $a0, $gp, $t6
    0x0000FC1C: 0xFA4800E0 '..H.' - sv.q       C200, 224($s2)
    0x0000FC20: 0xFA4C00F0 '..L.' - sv.q       C300, 240($s2)
    0x0000FC24: 0x14800079 'y...' - bnez       $a0, loc_0000FE0C
    0x0000FC28: 0x00142840 '@(..' - sll        $a1, $s4, 1
    0x0000FC2C: 0x00B45021 '!P..' - addu       $t2, $a1, $s4
    0x0000FC30: 0x00EAA021 '!...' - addu       $s4, $a3, $t2
    0x0000FC34: 0x264B0100 '..K&' - addiu      $t3, $s2, 256

loc_0000FC38:       ; Refs: 0x0000FC78 
    0x0000FC38: 0x264A0020 ' .J&' - addiu      $t2, $s2, 32

loc_0000FC3C:       ; Refs: 0x0000FC70 
    0x0000FC3C: 0x8E450004 '..E.' - lw         $a1, 4($s2)
    0x0000FC40: 0x8E430008 '..C.' - lw         $v1, 8($s2)
    0x0000FC44: 0x8E440000 '..D.' - lw         $a0, 0($s2)
    0x0000FC48: 0x8E42000C '..B.' - lw         $v0, 12($s2)
    0x0000FC4C: 0x0005E202 '....' - srl        $gp, $a1, 8
    0x0000FC50: 0x7CA4FE04 '...|' - ins        $a0, $a1, 24, 8
    0x0000FC54: 0x7C7CFC04 '..||' - ins        $gp, $v1, 16, 16
    0x0000FC58: 0x0003AC02 '....' - srl        $s5, $v1, 16
    0x0000FC5C: 0xAE840000 '....' - sw         $a0, 0($s4)
    0x0000FC60: 0x26520010 '..R&' - addiu      $s2, $s2, 16
    0x0000FC64: 0x7C55FA04 '..U|' - ins        $s5, $v0, 8, 24
    0x0000FC68: 0xAE9C0004 '....' - sw         $gp, 4($s4)
    0x0000FC6C: 0x2694000C '...&' - addiu      $s4, $s4, 12
    0x0000FC70: 0x164AFFF2 '..J.' - bne        $s2, $t2, loc_0000FC3C
    0x0000FC74: 0xAE95FFFC '....' - sw         $s5, -4($s4)
    0x0000FC78: 0x164BFFEF '..K.' - bne        $s2, $t3, loc_0000FC38
    0x0000FC7C: 0x26940018 '...&' - addiu      $s4, $s4, 24
    0x0000FC80: 0x26730001 '..s&' - addiu      $s3, $s3, 1

loc_0000FC84:       ; Refs: 0x0000FE68 0x0000FEB0 
    0x0000FC84: 0x0660FDAB '..`.' - bltz       $s3, loc_0000F334
    0x0000FC88: 0x0013E200 '....' - sll        $gp, $s3, 8
; Data ref 0x00119B70 ... 0x00000000 0x00000000 0x00000000 0x00000000 
    0x0000FC8C: 0x8E229B70 'p.".' - lw         $v0, -25744($s1)
    0x0000FC90: 0x24E40300 '...$' - addiu      $a0, $a3, 768
    0x0000FC94: 0x24E70200 '...$' - addiu      $a3, $a3, 512
    0x0000FC98: 0x004E5824 '$XN.' - and        $t3, $v0, $t6
    0x0000FC9C: 0x008B380A '.8..' - movz       $a3, $a0, $t3
    0x0000FCA0: 0x012CA02B '+.,.' - sltu       $s4, $t1, $t4
    0x0000FCA4: 0x00F8982B '+...' - sltu       $s3, $a3, $t8
    0x0000FCA8: 0x02939024 '$...' - and        $s2, $s4, $s3
    0x0000FCAC: 0x1640FDA0 '..@.' - bnez       $s2, loc_0000F330
    0x0000FCB0: 0x2413FFFA '...$' - li         $s3, -6

loc_0000FCB4:       ; Refs: 0x0000F2F8 
    0x0000FCB4: 0x0189C02B '+...' - sltu       $t8, $t4, $t1
    0x0000FCB8: 0x1300004D 'M...' - beqz       $t8, loc_0000FDF0
    0x0000FCBC: 0x012C682B '+h,.' - sltu       $t5, $t1, $t4
    0x0000FCC0: 0x01804821 '!H..' - move       $t1, $t4

loc_0000FCC4:       ; Refs: 0x0000FDF0 0x0000FDFC 
    0x0000FCC4: 0xCBAF0000 '....' - lv.s       S330, 0($sp)
    0x0000FCC8: 0xD00680B3 '....' - vzero.q    R403
    0x0000FCCC: 0xDD00C004 '....' - vpfxt      [x, y, 0, 0]
    0x0000FCD0: 0x6031B1B0 '..1`' - vadd.q     R400, R401, R401
; Data ref 0x00119B70 ... 0x00000000 0x00000000 0x00000000 0x00000000 
    0x0000FCD4: 0x262C9B70 'p.,&' - addiu      $t4, $s1, -25744
    0x0000FCD8: 0x8D9C0014 '....' - lw         $gp, 20($t4)
    0x0000FCDC: 0x8D850010 '....' - lw         $a1, 16($t4)
    0x0000FCE0: 0x8D8F0008 '....' - lw         $t7, 8($t4)
    0x0000FCE4: 0x013CA823 '#.<.' - subu       $s5, $t1, $gp
    0x0000FCE8: 0x00E81823 '#...' - subu       $v1, $a3, $t0
    0x0000FCEC: 0x00155043 'CP..' - sra        $t2, $s5, 1
    0x0000FCF0: 0x00A37023 '#p..' - subu       $t6, $a1, $v1
    0x0000FCF4: 0x01EA3823 '#8..' - subu       $a3, $t7, $t2
    0x0000FCF8: 0xAD890014 '....' - sw         $t1, 20($t4)
    0x0000FCFC: 0xAD8E0010 '....' - sw         $t6, 16($t4)
    0x0000FD00: 0xAD870008 '....' - sw         $a3, 8($t4)

loc_0000FD04:       ; Refs: 0x0000F2B8 
; Data ref 0x00119B70 ... 0x00000000 0x00000000 0x00000000 0x00000000 
    0x0000FD04: 0x262B9B70 'p.+&' - addiu      $t3, $s1, -25744
    0x0000FD08: 0x8D640010 '..d.' - lw         $a0, 16($t3)
; Data ref 0x00119B70 ... 0x00000000 0x00000000 0x00000000 0x00000000 
    0x0000FD0C: 0x8E349B70 'p.4.' - lw         $s4, -25744($s1)
    0x0000FD10: 0x3C120800 '...<' - lui        $s2, 0x800
    0x0000FD14: 0x00C43023 '#0..' - subu       $a2, $a2, $a0
    0x0000FD18: 0x00069840 '@...' - sll        $s3, $a2, 1
    0x0000FD1C: 0x02924024 '$@..' - and        $t0, $s4, $s2
    0x0000FD20: 0x02664821 '!Hf.' - addu       $t1, $s3, $a2
    0x0000FD24: 0x15000002 '....' - bnez       $t0, loc_0000FD30
    0x0000FD28: 0x00099040 '@...' - sll        $s2, $t1, 1
    0x0000FD2C: 0x00069080 '....' - sll        $s2, $a2, 2

loc_0000FD30:       ; Refs: 0x0000FD24 
; Data ref 0x00119B70 ... 0x00000000 0x00000000 0x00000000 0x00000000 
    0x0000FD30: 0x26319B70 'p.1&' - addiu      $s1, $s1, -25744
    0x0000FD34: 0x8E380014 '..8.' - lw         $t8, 20($s1)
    0x0000FD38: 0x00C02821 '!(..' - move       $a1, $a2
    0x0000FD3C: 0x02403021 '!0@.' - move       $a2, $s2
    0x0000FD40: 0x0C002CDE '.,..' - jal        sub_0000B378
    0x0000FD44: 0x03102023 '# ..' - subu       $a0, $t8, $s0
    0x0000FD48: 0x8E260010 '..&.' - lw         $a2, 16($s1)
    0x0000FD4C: 0x50C00023 '#..P' - beqzl      $a2, loc_0000FDDC
    0x0000FD50: 0x24040001 '...$' - li         $a0, 1
    0x0000FD54: 0x8E23000C '..#.' - lw         $v1, 12($s1)

loc_0000FD58:       ; Refs: 0x0000FDE4 
    0x0000FD58: 0x28790080 '..y(' - slti       $t9, $v1, 128
    0x0000FD5C: 0x17200017 '.. .' - bnez       $t9, loc_0000FDBC
    0x0000FD60: 0x00000000 '....' - nop        
    0x0000FD64: 0x8E2F0014 '../.' - lw         $t7, 20($s1)
    0x0000FD68: 0x01F08023 '#...' - subu       $s0, $t7, $s0
    0x0000FD6C: 0x00707023 '#pp.' - subu       $t6, $v1, $s0
    0x0000FD70: 0x19C0000B '....' - blez       $t6, loc_0000FDA0
    0x0000FD74: 0xAE2E000C '....' - sw         $t6, 12($s1)

loc_0000FD78:       ; Refs: 0x0000FDD4 
    0x0000FD78: 0x8FBF002C ',...' - lw         $ra, 44($sp)

loc_0000FD7C:       ; Refs: 0x0000FDB4 
    0x0000FD7C: 0x8FBC0028 '(...' - lw         $gp, 40($sp)
    0x0000FD80: 0x8FB50024 '$...' - lw         $s5, 36($sp)
    0x0000FD84: 0x8FB40020 ' ...' - lw         $s4, 32($sp)
    0x0000FD88: 0x8FB3001C '....' - lw         $s3, 28($sp)
    0x0000FD8C: 0x8FB20018 '....' - lw         $s2, 24($sp)
    0x0000FD90: 0x8FB10014 '....' - lw         $s1, 20($sp)
    0x0000FD94: 0x8FB00010 '....' - lw         $s0, 16($sp)
    0x0000FD98: 0x03E00008 '....' - jr         $ra
    0x0000FD9C: 0x27BD0030 '0..'' - addiu      $sp, $sp, 48

loc_0000FDA0:       ; Refs: 0x0000FD70 
    0x0000FDA0: 0x264AE800 '..J&' - addiu      $t2, $s2, -6144
    0x0000FDA4: 0x0150882C ',.P.' - max        $s1, $t2, $s0
    0x0000FDA8: 0x02202821 '!( .' - move       $a1, $s1
    0x0000FDAC: 0x0C002CCD '.,..' - jal        sub_0000B334
    0x0000FDB0: 0x00002021 '! ..' - move       $a0, $zr
    0x0000FDB4: 0x08003F5F '_?..' - j          loc_0000FD7C
    0x0000FDB8: 0x8FBF002C ',...' - lw         $ra, 44($sp)

loc_0000FDBC:       ; Refs: 0x0000FD5C 
    0x0000FDBC: 0x0C002BEA '.+..' - jal        sub_0000AFA8
    0x0000FDC0: 0x00002021 '! ..' - move       $a0, $zr
    0x0000FDC4: 0x8E270014 '..'.' - lw         $a3, 20($s1)
    0x0000FDC8: 0x8E2C000C '..,.' - lw         $t4, 12($s1)
    0x0000FDCC: 0x00F01023 '#...' - subu       $v0, $a3, $s0
    0x0000FDD0: 0x01826823 '#h..' - subu       $t5, $t4, $v0
    0x0000FDD4: 0x08003F5E '^?..' - j          loc_0000FD78
    0x0000FDD8: 0xAE2D000C '..-.' - sw         $t5, 12($s1)

loc_0000FDDC:       ; Refs: 0x0000FD4C 
    0x0000FDDC: 0x0C002CCD '.,..' - jal        sub_0000B334
    0x0000FDE0: 0x02402821 '!(@.' - move       $a1, $s2
    0x0000FDE4: 0x08003F56 'V?..' - j          loc_0000FD58
    0x0000FDE8: 0x8E23000C '..#.' - lw         $v1, 12($s1)

loc_0000FDEC:       ; Refs: 0x0000FE04 
    0x0000FDEC: 0x012C682B '+h,.' - sltu       $t5, $t1, $t4

loc_0000FDF0:       ; Refs: 0x0000FCB8 
    0x0000FDF0: 0x11A0FFB4 '....' - beqz       $t5, loc_0000FCC4
    0x0000FDF4: 0x3403FE00 '...4' - li         $v1, 0xFE00
    0x0000FDF8: 0x95390000 '..9.' - lhu        $t9, 0($t1)
    0x0000FDFC: 0x1723FFB1 '..#.' - bne        $t9, $v1, loc_0000FCC4
    0x0000FE00: 0x00000000 '....' - nop        
    0x0000FE04: 0x08003F7B '{?..' - j          loc_0000FDEC
    0x0000FE08: 0x25290002 '..)%' - addiu      $t1, $t1, 2

loc_0000FE0C:       ; Refs: 0x0000FC24 
; Data ref 0x00119B70 ... 0x00000000 0x00000000 0x00000000 0x00000000 
    0x0000FE0C: 0x8E239B70 'p.#.' - lw         $v1, -25744($s1)
    0x0000FE10: 0x000BA900 '....' - sll        $s5, $t3, 4
    0x0000FE14: 0x00F55821 '!X..' - addu       $t3, $a3, $s5
    0x0000FE18: 0x00035282 '.R..' - srl        $t2, $v1, 10
    0x0000FE1C: 0x7D4AFC04 '..J}' - ins        $t2, $t2, 16, 16
    0x0000FE20: 0x26450100 '..E&' - addiu      $a1, $s2, 256

loc_0000FE24:       ; Refs: 0x0000FE60 
    0x0000FE24: 0x26440020 ' .D&' - addiu      $a0, $s2, 32

loc_0000FE28:       ; Refs: 0x0000FE58 
    0x0000FE28: 0x8E5C0000 '..\.' - lw         $gp, 0($s2)
    0x0000FE2C: 0x8E420004 '..B.' - lw         $v0, 4($s2)
    0x0000FE30: 0x26520008 '..R&' - addiu      $s2, $s2, 8
    0x0000FE34: 0x7F9C50C4 '.P..' - ins        $gp, $gp, 3, 8
    0x0000FE38: 0x7C4250C4 '.PB|' - ins        $v0, $v0, 3, 8
    0x0000FE3C: 0x7F9C90C4 '....' - ins        $gp, $gp, 3, 16
    0x0000FE40: 0x7C4290C4 '..B|' - ins        $v0, $v0, 3, 16
    0x0000FE44: 0x001CAA43 'C...' - sra        $s5, $gp, 9
    0x0000FE48: 0x7EAA7004 '.p.~' - ins        $t2, $s5, 0, 15
    0x0000FE4C: 0x0002E243 'C...' - sra        $gp, $v0, 9
    0x0000FE50: 0x7F8AF404 '....' - ins        $t2, $gp, 16, 15
    0x0000FE54: 0xAD6A0000 '..j.' - sw         $t2, 0($t3)
    0x0000FE58: 0x1644FFF3 '..D.' - bne        $s2, $a0, loc_0000FE28
    0x0000FE5C: 0x256B0004 '..k%' - addiu      $t3, $t3, 4
    0x0000FE60: 0x1645FFF0 '..E.' - bne        $s2, $a1, loc_0000FE24
    0x0000FE64: 0x256B0010 '..k%' - addiu      $t3, $t3, 16
    0x0000FE68: 0x08003F21 '!?..' - j          loc_0000FC84
    0x0000FE6C: 0x26730001 '..s&' - addiu      $s3, $s3, 1

loc_0000FE70:       ; Refs: 0x0000F824 
    0x0000FE70: 0xFA540000 '..T.' - sv.q       C500, 0($s2)
    0x0000FE74: 0xFA440010 '..D.' - sv.q       C100, 16($s2)
    0x0000FE78: 0xFA580020 ' .X.' - sv.q       C600, 32($s2)
    0x0000FE7C: 0xFA5C0030 '0.\.' - sv.q       C700, 48($s2)
    0x0000FE80: 0xFA550040 '@.U.' - sv.q       C510, 64($s2)
    0x0000FE84: 0xFA450050 'P.E.' - sv.q       C110, 80($s2)
    0x0000FE88: 0xFA590060 '`.Y.' - sv.q       C610, 96($s2)
    0x0000FE8C: 0xFA5D0070 'p.].' - sv.q       C710, 112($s2)
    0x0000FE90: 0xFA560080 '..V.' - sv.q       C520, 128($s2)
    0x0000FE94: 0xFA460090 '..F.' - sv.q       C120, 144($s2)
    0x0000FE98: 0xFA5A00A0 '..Z.' - sv.q       C620, 160($s2)
    0x0000FE9C: 0xFA5E00B0 '..^.' - sv.q       C720, 176($s2)
    0x0000FEA0: 0xFA5700C0 '..W.' - sv.q       C530, 192($s2)
    0x0000FEA4: 0xFA4700D0 '..G.' - sv.q       C130, 208($s2)
    0x0000FEA8: 0xFA5B00E0 '..[.' - sv.q       C630, 224($s2)
    0x0000FEAC: 0xFA5F00F0 '.._.' - sv.q       C730, 240($s2)
    0x0000FEB0: 0x08003F21 '!?..' - j          loc_0000FC84
    0x0000FEB4: 0x26730001 '..s&' - addiu      $s3, $s3, 1

loc_0000FEB8:       ; Refs: 0x0000F37C 
    0x0000FEB8: 0x44820000 '...D' - mtc1       $v0, $fcr0
    0x0000FEBC: 0x3C010001 '...<' - lui        $at, 0x1
    0x0000FEC0: 0xC4273400 '.4'.' - lwc1       $fpr07, 13312($at)
    0x0000FEC4: 0x46800220 ' ..F' - cvt.s.w    $fpr08, $fpr00
    0x0000FEC8: 0x166DFD3A ':.m.' - bne        $s3, $t5, loc_0000F3B4
    0x0000FECC: 0x46074042 'B@.F' - mul.s      $fpr01, $fpr08, $fpr07
    0x0000FED0: 0x64121C1C '...d' - vmul.s     S700, S700, S420
    0x0000FED4: 0x08003CED '.<..' - j          loc_0000F3B4
    0x0000FED8: 0x46020842 'B..F' - mul.s      $fpr01, $fpr01, $fpr02
    0x0000FEDC: 0x27BDFFF0 '...'' - addiu      $sp, $sp, -16
    0x0000FEE0: 0x3C030001 '...<' - lui        $v1, 0x1
    0x0000FEE4: 0xAFB00000 '....' - sw         $s0, 0($sp)
    0x0000FEE8: 0x00808021 '!...' - move       $s0, $a0
    0x0000FEEC: 0x02002821 '!(..' - move       $a1, $s0
    0x0000FEF0: 0xAFBF0004 '....' - sw         $ra, 4($sp)
    0x0000FEF4: 0x0C004297 '.B..' - jal        sub_00010A5C
; Data ref 0x0000FEDC ... 0x27BDFFF0 0x3C030001 0xAFB00000 0x00808021 
    0x0000FEF8: 0x2464FEDC '..d$' - addiu      $a0, $v1, -292
    0x0000FEFC: 0x3C040012 '...<' - lui        $a0, 0x12
    0x0000FF00: 0x04400005 '..@.' - bltz       $v0, loc_0000FF18
; Data ref 0x00119B8C ... 0x00000000 0x00000000 0x00000000 0x00000000 
    0x0000FF04: 0x24839B8C '...$' - addiu      $v1, $a0, -25716

loc_0000FF08:       ; Refs: 0x0000FF4C 
    0x0000FF08: 0x8FBF0004 '....' - lw         $ra, 4($sp)

loc_0000FF0C:       ; Refs: 0x0000FF5C 
    0x0000FF0C: 0x8FB00000 '....' - lw         $s0, 0($sp)
    0x0000FF10: 0x03E00008 '....' - jr         $ra
    0x0000FF14: 0x27BD0010 '...'' - addiu      $sp, $sp, 16

loc_0000FF18:       ; Refs: 0x0000FF00 
    0x0000FF18: 0x90680130 '0.h.' - lbu        $t0, 304($v1)
    0x0000FF1C: 0x8C65012C ',.e.' - lw         $a1, 300($v1)
    0x0000FF20: 0x24040002 '...$' - li         $a0, 2
    0x0000FF24: 0x35070020 ' ..5' - ori        $a3, $t0, 0x20
    0x0000FF28: 0x30E6007F '...0' - andi       $a2, $a3, 0x7F
    0x0000FF2C: 0x10A00002 '....' - beqz       $a1, loc_0000FF38
    0x0000FF30: 0xA0660130 '0.f.' - sb         $a2, 304($v1)
    0x0000FF34: 0xACA00000 '....' - sw         $zr, 0($a1)

loc_0000FF38:       ; Refs: 0x0000FF2C 
    0x0000FF38: 0xAC70012C ',.p.' - sw         $s0, 300($v1)
    0x0000FF3C: 0xA0600033 '3.`.' - sb         $zr, 51($v1)
    0x0000FF40: 0x906A0131 '1.j.' - lbu        $t2, 305($v1)
    0x0000FF44: 0x92090004 '....' - lbu        $t1, 4($s0)
    0x0000FF48: 0x012A2824 '$(*.' - and        $a1, $t1, $t2
    0x0000FF4C: 0x10A0FFEE '....' - beqz       $a1, loc_0000FF08
    0x0000FF50: 0xA0690132 '2.i.' - sb         $t1, 306($v1)
    0x0000FF54: 0x0C002B5E '^+..' - jal        sub_0000AD78
    0x0000FF58: 0x00000000 '....' - nop        
    0x0000FF5C: 0x08003FC3 '.?..' - j          loc_0000FF0C
    0x0000FF60: 0x8FBF0004 '....' - lw         $ra, 4($sp)
```