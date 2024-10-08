# Introduction

The repository of NEXST (Next Environment for XiangShan Target) 
contains basic hardware and software components 
for system-level FPGA prototyping and emulation 
of the open-source XiangShan RISC-V processor core.

## XiangShan Targets with Nanhu Microarchitecture
- **Nanhu-G**, a compact version of XiangShan Nanhu   

- **Nanhu-minimal**, a minimal version of XiangShan NanHu

Note: Please refer to https://xiangshan-doc.readthedocs.io 
for more detailed information of Xiangshan Nanhu microarchitecture

## Fully-fledged and Cost-effective FPGA Environment
- **AMD/Xilinx VCU128**, a commercial development board with 
the AMD/Xilinx Ultrascale+ VU37P FPGA chip
(https://www.xilinx.com/products/boards-and-kits/vcu128.html) 

- **NM37**, a custom acceleration card designed by our team, 
allowing integration of a PCIe-attached NVMe SSD to XiangShan SoC   

# Prerequisite

1. Download all required repository submodules

`git submodule update --init --recursive`   

2. Launch the following commands

`mkdir -p work_farm/target/`    
`cd work_farm/target && ln -s ../../nanhu-g nanhu-g`   
`cd ../ && ln -s ../shell shell`   
`ln -s ../tools/ tools` 

3. Install Linux kernel header on the x86 server 
side where the FPGA board/card is attached   

4. Install `minicom` on the x86 server  

5. Prepare the Chisel compilation environment on the x86 server  

5. All compilation operations are launched in the directory of `work_farm`

# Hardware Generation (including Nanhu-G core and its SoC wrapper)

## Nanhu-G compilation

`make PRJ="target:nanhu-g" FPGA_BD=vcu128 xs_gen`

## FPGA design flow

### FPGA Wrapper generation   
`make PRJ="shell:vcu128" FPGA_BD=vcu128 FPGA_ACT=prj_gen vivado_prj`    
`make PRJ="shell:vcu128" FPGA_BD=vcu128 FPGA_ACT=run_syn vivado_prj`   

### Xiangshan FPGA synthesis  
`make PRJ="target:nanhu-g:proto" FPGA_BD=vcu128 FPGA_ACT=prj_gen vivado_prj`   
`make PRJ="target:nanhu-g:proto" FPGA_BD=vcu128 FPGA_ACT=run_syn vivado_prj`

### FPGA Bitstream generation  
`make PRJ="target:nanhu-g:proto" FPGA_BD=vcu128 FPGA_ACT=bit_gen vivado_prj`

    The bitstream file is located in   
    `nanhu-g/ready_for_download/proto_vcu128/`
    
    Log files, timing and utilization reports and 
    design checkpoint files (.dcp) generated during Xilinx Vivado design flow 
    are located in   
    `work_farm/fpga/vivado_out/target_nanhu-g_proto_vcu128/` 
    
    Generated Vivado project of the SoC wrapper and target XiangShan 
    are located in  
    `work_farm/fpga/vivado_prj/shell_vcu128_vcu128/` and   
    `work_farm/fpga/vivado_prj/target_nanhu-g_proto_vcu128`, respectively.   

**If you want to deploy prototyping on the NM37 card, please 
substitute the `vcu128` to `nm37_vu37p` in each command line.** 

# RISC-V Side Software Compilation

## Compilation of ZSBL image leveraged in Boot ROM

`make PRJ="target:nanhu-g:proto" FPGA_BD=vcu128 ARCH=riscv zsbl`   

    The bootrom.bin is located in
    `nanhu-g/ready_for_download/proto_vcu128/`

## Linux boot via OpenSBI

If you want to deploy prototyping on the NM37 card with NVMe SSD, please 
substitute the value of `DT_TARGET` from `XSTop` to `XSTop_pci` in the following command line.

### DTB generation
`make PRJ="target:nanhu-g:proto" FPGA_BD=vcu128 DT_TARGET=XSTop target_dt` 

If you want to generate the device tree blob that specifies the PCIe root port, please substitute the `DT_TARGET` variable by `XSTop_pci`   

### Linux kernel (v5.16) compilation
`make PRJ="target:nanhu-g:proto" FPGA_BD=vcu128 ARCH=riscv phy_os.os`   

### OpenSBI compilation (RV_BOOT.bin generation)
`make PRJ="target:nanhu-g:proto" FPGA_BD=vcu128 ARCH=riscv DT_TARGET=XSTop opensbi`   

    The boot image (i.e., RV_BOOT.bin) is located in
    `nanhu-g/ready_for_download/proto_vcu128/`

# FPGA evaluation flow

## Setup of Hardware Environment

- Attach one of the target FPGA boards listed above 
  to a PCIe slot of one x86 host machine. 
  
  Please carefully check if the power supply of the PCIe-attached FPGA board is properly setup.

  AMD/Xilinx Vivado toolset must be installed on the x86 host.

## Compilation of x86-side PCIe XDMA Linux driver

### Clone this repository on the x86 host and update the submodule of shell/software/xdma_drv

### Driver compilation

`make PRJ="shell:vcu128" xdma_drv`   

    The kernel module (i.e., xdma.ko) is located in
    `shell/software/build/xdma_drv/`

## Evaluation steps

- Ensure that `bootrom.bin`, `RV_BOOT.bin` mentioned in the above steps   
  and `tools/proto` directory have been generated or located on the x86 host machine.   

- Use Vivado toolset to program the FPGA with `system.bit` generated in the steps above   
(in `nanhu-g/ready_for_download/proto_vcu128/`).

- Restart the x86 host machine to probe the FPGA as a PCIe device.

- Load XDMA driver.

    `cd shell/software/build/xdma_drv && sudo insmod xdma.ko`

    If successful, a series of `/dev/xdma<N>*` devices will be created (`<N>` is an assigned number), and detailed log can be viewed in `sudo dmesg`.

- Open minicom to setup a console as the XiangShan terminal.  
    Please launch `sudo minicom -s /dev/xdma0_ttyUL0` to configure the terminal as 115200n8    

- Load images & run.

    Launch the following commands:
    ```sh
    cd tools/proto
    make # if proto is not compiled
    sudo ./load_and_run.sh xdma<N> bootrom.bin RV_BOOT.bin # <N> is the assigned xdma device number
    ```

    This will reset the XiangShan core, load images and start the execution. At the end the serial is connected and the user can interact with the system running on XiangShan. To exit the serial connection, press the escape key CTRL+A+X.

    To resume the serial connection without a system reset, use the following command:

    ```sh
    sudo ./load_and_run.sh xdma<N>
    ```