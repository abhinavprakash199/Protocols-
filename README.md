# Protocols-
This repository contains the whole summary of hands on done by Abhinav Prakash (IS22MTECH14002) on the following Protocols 





## UART(Universal asynchronous receiver-transmitter)  Protocols
---
### Verilog Codes 
```verilog
`timescale 1ns / 1ps

module UART(
    input clk,
    input start,
    input [7:0] txin,
    output reg tx, 
    input rx,
    output [7:0] rxout,
    output rxdone, txdone
    );
    
 parameter clk_value = 100_000;
 parameter baud = 9600;
 
 parameter wait_count = clk_value / baud;
 
 reg bitDone = 0;
 integer count = 0;
 parameter idle = 0, send = 1, check = 2;
 reg [1:0] state = idle;
 
///////////////////Generate Trigger for Baud Rate
 always@(posedge clk)
 begin
  if(state == idle)
    begin 
    count <= 0;
    end
  else 
    begin
      if(count == wait_count)
        begin
          bitDone <= 1'b1;
          count   <= 0;  
        end
      else
        begin
          count   <= count + 1;
          bitDone <= 1'b0;  
        end    
    end
 end
 
                                                                   //TX Logic
                                                                   
 reg [9:0] txData;///stop bit data start
 integer bitIndex = 0; ///reg [3:0];
 reg [9:0] shifttx = 0;
 
 
 always@(posedge clk)
 begin
     case(state)
     idle : 
         begin
         tx       <= 1'b1;
         txData   <= 0;
         bitIndex <= 0;
         shifttx  <= 0;
               
            if(start == 1'b1)
               begin
                 txData <= {1'b1,txin,1'b0};
                 state  <= send;
               end
            else
               begin           
                 state <= idle;
               end
         end
 
     send: 
          begin
            tx       <= txData[bitIndex];
            state    <= check;
            shifttx  <= {txData[bitIndex], shifttx[9:1]};
          end 
      
     check: 
        begin      
          if(bitIndex <= 9) ///0 - 9 = 10
            begin
               if(bitDone == 1'b1)
                   begin
                     state <= send;
                     bitIndex <= bitIndex + 1;
                   end
            end
          else
            begin
              state <= idle;
              bitIndex <= 0;
            end
         end
     
    default: state <= idle;
    endcase
 end
 
assign txdone = (bitIndex == 9 && bitDone == 1'b1) ? 1'b1 : 1'b0;
 
 
                                                             //RX Logic
 
 integer rcount = 0;
 integer rindex = 0;
 parameter ridle = 0, rwait = 1, recv = 2;
 reg [1:0] rstate;
 reg [9:0] rxdata;
 always@(posedge clk)
 begin
     case(rstate)
     ridle : 
        begin
          rxdata <= 0;
          rindex <= 0;
          rcount <= 0;
          if(rx == 1'b0)
             begin
               rstate <= rwait;
             end
           else
             begin
               rstate <= ridle;
             end
        end
         
     rwait : 
        begin
          if(rcount < wait_count / 2)
             begin
               rcount <= rcount + 1;
               rstate <= rwait;
             end
          else
             begin
               rcount <= 0;
               rstate <= recv;
               rxdata <= {rx,rxdata[9:1]}; 
             end
        end
     
     
     recv : 
        begin
          if(rindex <= 9) 
             begin
               if(bitDone == 1'b1) 
                  begin
                    rindex <= rindex + 1;
                    rstate <= rwait;
                  end
             end
          else
             begin
               rstate <= ridle;
               rindex <= 0;
             end
        end
     
     default : rstate <= ridle;
     endcase
 end
 
 
assign rxout = rxdata[8:1]; 
assign rxdone = (rindex == 9 && bitDone == 1'b1) ? 1'b1 : 1'b0;
 
 
 endmodule
 
```

### Testbench 

```verilog
`timescale 1ns / 1ps

module tb;
 
    reg clk = 0;
    reg start = 0;
    reg [7:0] txin;
    wire [7:0] rxout;
    wire rxdone, txdone;
 
    wire txrx;
    integer i = 0;
    
 UART dut (clk, start, txin, txrx,txrx, rxout, rxdone, txdone );
    
 
 initial 
    begin
      start = 1;
      for(i = 0; i < 10; i = i + 1) 
         begin
           txin = $urandom_range(10 , 200);
           @(posedge rxdone);
           @(posedge txdone);
         end
      $stop; 
    end
 
 always #5 clk = ~clk;
 
 endmodule
```

### Simulation Results 

![Screenshot (2578)](https://user-images.githubusercontent.com/120498080/233801036-b54882e6-d105-440f-9965-ee7d8aba1d0e.png)

### Designing IP and Testing
- Then I created an IP(Named - UART_IP_NEW2) of our design and tested it UART Communication

![image](https://github.com/abhinavprakash199/Protocols-/assets/120498080/507c784b-2eff-43c4-8e74-939ffe620e1a)

- Then using SDK I directly provided TX bit at the assigned memory 
- I give bit as 155 in hex which is `000101010101` in binary in the 32 bit slave register (slv_reg0), where LSB is for the start bit which is always high to transmit the tx data and next 8 bit after LSB is for tx data which is `10101010` in binary and 170 in decimal.
- For the output side I define another slave register (slv_reg1) of 32 bit in which I will save the received data, in that I consider 8 bit from LSB as the RX bit, 9th bit as rxdone, 10th bit as txdone.
- So for proper working to the UART, I need to get RX bit as `1010101010` and rxdone and tx done as high, e.g. - slv_reg1 should have 3AA in hex and `1110101010` in binary.
- In my IP slv_reg0 can be accessed with address at `XPAR_UART_IP_NEW2_0_S00_AXI_BASEADDR` and slv_reg1 is 4 bit after this address.
- So I finally verify the UART protocal using using ZYNQ FPGA boart and FT232RL uart to usb ic.

![image](https://github.com/abhinavprakash199/Protocols-/assets/120498080/81b6f53a-feda-4d7b-8c26-1723cbd99067)
   



