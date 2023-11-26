# Protocols
This repository contains the whole summary of hands on done by Abhinav Prakash (IS22MTECH14002) on the following Protocols 

# Table of Content
---

* [UART(Universal asynchronous receiver-transmitter) Protocols](#UART(Universal-asynchronous-receiver-transmitter)-Protocols)
    + [Verilog Codes](#Verilog-Codes)
    + [Testbench](#Testbench)
    + [Simulation Results](#Simulation-Results)
    + [Designing IP and Testing](#Designing-IP-and-Testing)
 
* [IIC(Inter-Integrated Circuit) Protocols](#IIC(Inter-Integrated-Circuit)-Protocols)
    + [Verilog Codes of Master](#Verilog-Codes-of-Master)
    + [Verilog Codes of Slave](#Verilog-Codes-of-Slave)
    + [Testbench](#Testbench)
    + [Simulation Results](#Simulation-Results)




## UART(Universal asynchronous receiver-transmitter) Protocols
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


## IIC(Inter-Integrated Circuit) Protocols
---
### Verilog Codes of Master
---
```verilog
`timescale 1ns / 1ps


module I2C_M1(
  
 input clk,
 input rst,
 input newd,
 input wr,
 input [7:0] wdata,
 input [6:0] addr, /////8-bit  7-bit : addr 1-bit : mode
   
 input sda_i,  
 output reg sda_t,  // if sda_t=0 then sda_o is on if sda_t=1 then sda_i is on 
 output reg sda_o,
 
 output scl,
 output reg [7:0] rdata,
 output reg done
 );
  
 reg x,x2,x3; 
 reg sclt, donet; 
 reg [7:0] rdatat; 
 reg [7:0] addrt;
 reg [3:0] state,nstate;
 reg sclk_ref = 0;
 integer count = 0;
 integer i = 0;
  
 parameter idle = 0, check_wr = 1, wstart = 2, wsend_addr = 3, waddr_ack = 4, 
                wsend_data = 5, wdata_ack = 6, wstop = 7, rsend_addr = 8, 
                raddr_ack = 9, rsend_data = 10,
                rdata_ack = 11, rstop = 12 ;
  
 always@(posedge clk)
    begin
      if(count <= 9) 
        begin
            count <= count + 1;     
        end
      else
        begin
            count     <= 0; 
            sclk_ref  <= ~sclk_ref;
        end	      
    end
  
  
  
  always@(posedge sclk_ref)
    begin 
      if(rst == 1'b1)
         begin
           state <= idle;
           sclt  <= 1'b1;
           sda_t  <= 1'b0;   // send sda = 0 as default
           sda_o <= 1'b1;
           donet <= 1'b0;
         end
       else 
            state <= nstate;
    end
    
    
  always@(posedge sclk_ref)  
    begin
        if (state ==  wsend_addr)
         if(i <= 7) 
              i = i + 1;
          
       
        if (state ==  waddr_ack)
           if(x == 1'b1)
              i = i + 1;
        if (state == wsend_data) 
            if(i <= 7) 
              i <= i + 1;
        if (state ==  rsend_addr)
           if(i <= 7) 
              i = i + 1;
        if (state == rsend_data)
            if(i <= 7) 
              i <= i + 1;      
          
    end      
          
            
  always@(*)          
         begin
         case(state)
           idle :  begin //state 0
                      done <= 1'b0;
                      sclt <= 1'b1;
                      sda_t <= 1'b0; //send sda = 1 as start bit
                      sda_o <=1'b1;
                        if(newd == 1'b1) 
                            nstate  <= wstart;
                        else 
                            nstate <= idle;         
                   end
         
           wstart: begin // state 2
                      sda_t  <= 1'b0; //send sda = 0 as start bit
                      sda_o  <= 1'b0;
                      sclt  <= 1'b1;
                      nstate <= check_wr;
                      addrt <= {addr,wr};
                    end
            
           check_wr: begin // state 1 
                        if(wr == 1'b1)  // wr = 1 write data
                         begin
                            nstate <= wsend_addr;
                            sda_t  <= 1'b0;   // to send 7 bit address and wr bit
                            sda_o <= addrt[0];
                            i <= 1;
                         end
                        else        //wr =0 read data
                         begin
                            nstate <= rsend_addr;
                            sda_t  <= 1'b0;   // to send 7 bit address and wr bit
                            sda_o <= addrt[0];
                            i <= 1;
                         end
                    end
            
           wsend_addr : begin  // state 3 
                        //i <= 1'b1;
                         if(i <= 7) 
                              begin
                                sda_o  <= addrt[i];
                                //i <= i + 1;
                              end
                            else
                              begin
                                //i <= 0;
                                nstate <= waddr_ack;
                                //sda_t  <= 1'b1; // to receive ack bit for add received
                                    if(sda_i == 1'b0)
                                        x= 1;
                                    else //if (sda_i == 1'b1)
                                        x=0; 
                              end   
                        end
             
           waddr_ack : begin  //state 4
                         i <= 1;
                         sda_t  <= 1'b1; // to receive ack bit for add received
                         if(x == 1) 
                            begin
                                nstate <= wsend_data;
                                sda_t  <= 1'b0;   // to send 8 bit data 
                                sda_o  <= wdata[0];
                                //i <= i + 1;
                            end
                         else
                           nstate <= waddr_ack;
                       end
         
           wsend_data : begin// state 5
                         if(i <= 7) 
                            begin
                               sda_o  <= wdata[i]; 
                               //i <= i + 1;
                               sda_t  <= 1'b0;
                             end
                         else 
                             begin
                                //i <= 0;
                                nstate <= wdata_ack;
                                sda_t  <= 1'b1;   // to receive ack bit for data received
                                     if (sda_i==0)
                                         x2= 1;
                                     else 
                                         x2=0;
                                end
                         end
                         
         wdata_ack :  begin //STATE 6
                       i <=0;
                        if(x2 == 1'b1) 
                            begin
                                nstate <= wstop;
                                sda_t  <= 1'b0;     // send sda = 0 as stop bit
                                sda_o <= 1'b0;
                                sclt <= 1'b1;
                            end
                         else 
                            begin
                                nstate <= wdata_ack;
                            end 
                       end
         
         wstop: begin
                      sda_t  <= 1'b0;   // send sda = 1 as stop bit
                      sda_o  <=  1'b1;
                      sclt  <=  1'b1;
                      nstate <=  idle;
                      done  <=  1'b1;  
                end
         
         ///////////////////////read state

         rsend_addr : begin
                        if(i <= 7) 
                            begin
                              sda_t  <= 1'b0;   // to send 7 bit address and wr bit
                              sda_o  <= addrt[i];
                              //i <= i + 1;
                            end
                        else
                            begin
                              i <= 0;
                              nstate <= raddr_ack;
                              sda_t <= 1'b1; // to receive ack bit for add received
                                if (sda_i ==0)
                                    x3 = 1;
                                else
                                    x3 = 0;
                            end   
                    end
         
           raddr_ack : begin   //state 9
                        if(x3 == 1'b1) 
                            begin
                                nstate  <= rsend_data;
                                sda_t <= 1'b1; // read incoming 8 bit date 
                                sda_o <= 1'b0;
                                rdata[0] <= sda_i;
                                i <= 1;
                            end
                        else
                            nstate <= raddr_ack;
                       end
                 
           rsend_data : begin  // state 10
                            if(i <= 7) 
                              begin
                                 //i <= i + 1;
                                 nstate <= rsend_data;
                                 rdata[i] <= sda_i;
                              end
                             else
                              begin
                                 i <= 0;
                                 nstate <= rdata_ack;
                                 sda_t  <= 1'b0;  // send ack bit
                                 sda_o <= 1'b0;  
                              end         
                            end
          
           rdata_ack : begin // state 11
                            nstate <= rstop;
                            sda_t  <= 1'b0;  // send sda = 0 as stop bit
                            sda_o <= 1'b0;  
                            sclt <= 1'b1;
                        end 
                           
           rstop:  begin   // state 12
                        sda_t  <= 1'b0; //send sda = 1 as stop bit
                        sda_o  <=  1'b1;
                        sclt <=  1'b1;
                        nstate <=  idle;
                        done  <=  1'b1;  
                    end
                    
           default : nstate <= idle;
        endcase
        end
 
  
 assign scl = (( state == wstart) || ( state == wstop) || ( state == rstop)) ? sclt : sclk_ref;
 
endmodule
```

### Testbench
---
```verilog
`timescale 1ns / 1ps


module TB_I2C_M();
    
 reg clk;
 reg rst;
 reg newd;
 //reg ack;
 reg wr;
 reg [7:0] wdata;
 reg [6:0] addr; /////8-bit  7-bit : addr 1-bit : mode
   
 reg sda_i;
 wire sda_t;
 wire sda_o;
 
 wire scl;
 wire [7:0] rdata;
 wire done;
 
 
 I2C_M1 DUT(.clk(clk),
 .rst(rst),
 .newd(newd),
 //.ack(ack),
 .wr(wr),
 .wdata(wdata),
 .addr(addr), /////8-bit  7-bit : addr 1-bit : mode
 .sda_i(sda_i),
 .sda_t(sda_t),
 .sda_o(sda_o),
 .scl(scl),
 .rdata(rdata),
 .done(done)
  );
  
 initial begin
 clk = 0;
 forever #10 clk =~clk;
 end
 
 // Reset generation
  initial begin
    rst = 1;
    #50 rst = 0;    
  end
  
  initial begin
    #60 newd = 1;
    #60 wr = 1;
    
  end
  
  initial begin
   #5050 sda_i = 0;
   #440 sda_i = 1;
   //#440 sda_i = 0;
   
  end
  initial begin
   #9010 sda_i = 0;
   #440 sda_i = 1;
  end
  initial begin
  #14730 sda_i = 0;
  #440 sda_i = 1;
  end
  
  //initial begin
  //sda_i = 1;
  //end
  
  initial begin
    #10330 wr = 0;
  end
  
  
  initial begin
    addr = 7'b1010101;
    wdata = 8'b11101010;
  end
 
 
 initial begin
    #15170 sda_i = 0;
    #440 sda_i = 1;
    #440 sda_i = 0;
    #440 sda_i = 1;
    #440 sda_i = 1;
    #440 sda_i = 0;
    #440 sda_i = 1;
    #440 sda_i = 0;
    //#440 sda_i = 1;
    
  end
endmodule
```

### Verilog Codes of Slave
---
```verilog
`timescale 1ns / 1ps

module I2C_S2(


 input scl,
 input rst,
 input newd_s,
 output reg wr,
 output reg [7:0] r_data,
 output reg [6:0] r_addr, /////8-bit  7-bit : addr 1-bit : mode
   
 input sda_i,  
 output reg sda_t,
 output reg sda_o,
 
 output reg [7:0] rdata,
 output reg done
 );
  
 //reg x,x2,x3; 
 reg donet; 
 reg y = 1'b0;
 reg [7:0] r_datat; 
 reg [7:0] addrt;
 reg [3:0] state,nstate;
 reg sclk_ref = 0;
 integer count = 0;
 integer i = 0;
 
 parameter slave_address = 7'b1000101;
 parameter idle = 0, check_wr = 1, wstart = 2, wsend_addr = 3, waddr_ack = 4, 
                wsend_data = 5, wdata_ack = 6, wstop = 7, rsend_addr = 8, 
                raddr_ack = 9, rsend_data = 10,
                rdata_ack = 11, rstop = 12 ;
  
  always@(*)
    begin
        case(state)
           idle : begin //state 0
                    if(newd_s == 1'b1)
                       done <= 1'b0;
                       sda_t <= 1'b1; //sda_i working to receive start bit
                            if (y == 0 && sda_i == 1 && scl == 1)
                                state  <= wstart;
                            else 
                                state <= idle;         
                  end
         
           wstart: begin // state 2
                     sda_t  <= 1'b1; //sda_i working to receive start bit
                      if (y ==0 && sda_i == 0 && scl ==1) 
                         state <= check_wr;
                      else 
                         state <= wstart;
                    end
           //default : state <= idle;          
        //endcase 
        //end                 
    
  //always@(*)
//begin
        //case(state)
           check_wr:  begin// state 1 
                        sda_t  <= 1'b1;   //sda_i working in recive wead write bit
                        y = 1'bZ;
                           if(sda_i == 1'b1) 
                             begin
                                nstate <= wsend_addr;
                                wr <= 1'b1;     // write data
                                i <= 1;
                             end
                            else 
                             begin
                                nstate <= wsend_addr;
                                wr <= 1'b0;    // read data
                                i <= 1;
                             end
                      end
            
            wsend_addr :begin  // state 3 
                          sda_t  <= 1'b1;   //sda_i working in recive addreess        
                            if(i <= 7) 
                              begin
                                addrt[i] <= sda_i;
                                nstate <= wsend_addr;
                              end
                            else
                              begin
                                i <= 0;
                                nstate <= waddr_ack;
                                wr <= addrt[0];
                                r_addr <= addrt[7:1];
                              end 
                        end  
                              
            waddr_ack : begin//state 4
                            if(r_addr == slave_address) 
                                begin
                                    sda_t  <= 1'b0; //sda_o working in send ack bit for address
                                    sda_o <= 1'b0;
                                    nstate <= wsend_data;
                                 end
                            else
                               nstate <= waddr_ack;
                        end
           wsend_data : begin // state 5
                            sda_t  <= 1'b1;   //sda_i working in recive data
                                if(i <= 7) 
                                begin
                                    r_datat[i] <= sda_i; 
                                    i <= i + 1;
                                    sda_o <= 1'b1;
                                end
                            else 
                                begin
                                    i <= 0;
                                    nstate <= wdata_ack;
                                    r_data <= r_datat;
                                    sda_t  <= 1'b0; //sda_o working in send ack bot for data
                                    sda_o <= 1'b0;
                                    y =1'b1;
                                end
                        end
           wdata_ack : begin //STATE 6
                        if(y == 1 && sda_i == 1'b0 && scl == 1'b1) 
                            begin
                                nstate <= wstop;
                                sda_t  <= 1'b1; //sda_i working to receive stop bit
                            end
                         else 
                            nstate <= wdata_ack;
                        end
         
           wstop: begin//state 7
                    if(y == 1 && sda_i == 1'b1 && scl == 1'b1)
                        begin
                        nstate <=  idle;
                        sda_t  <= 1'b1; //sda_i working to receive stop bit
                        done  <=  1'b1;
                        end
                    else
                        nstate <= wstop;  
                end
           default : nstate <= idle;
      endcase
   end
  
  
  
  always@(posedge scl)
    begin 
    if(rst == 1'b1)
         begin
           state <= idle;
           sda_t  <= 1'b1;   //sda_i working in dafault
           sda_o <= 1'b1;
           donet <= 1'b0;
         end
       else 
           state <= nstate;
    end
           
       
  always@(posedge scl)
         begin
         if (state == wsend_addr)
            if(i <= 7) 
                i <= i + 1;
         if (state == waddr_ack)
            if(r_addr == slave_address)
                i <= i + 1;
         if (state == wsend_data)
            if(i <= 7) 
                i <= i + 1;
          end
    
endmodule
```

### Testbench
---
```verilog
`timescale 1ns / 1ps

module TB_I2C_S();

 reg clk;
 
 reg scl;
 reg rst;
 reg newd_s;
 //reg ack;
 wire wr;
 wire [7:0] r_data;
 wire [6:0] r_addr; /////8-bit  7-bit : addr 1-bit : mode
   
 reg sda_i;
 wire sda_t;
 wire sda_o;
 
 
 wire [7:0] rdata;
 wire done;
 
 reg [7:0] addressToSend 	= 8'b10001011;  //LSB is rw bin and rest 7 is address
 reg [7:0] dataToSend 	= 8'b10001011;
 reg sclt;
 integer j = 0,k = 0;
 
 I2C_S2 DUT(.scl(scl),
 .rst(rst),
 .newd_s(newd_s),
 .wr(wr),
 .r_data(r_data),
 .r_addr(r_addr), /////8-bit  7-bit : addr 1-bit : mode
 .sda_i(sda_i),
 .sda_t(sda_t),
 .sda_o(sda_o),
 .rdata(rdata),
 .done(done)
  );

initial begin
//sda_t = 1'b1;
     rst = 1;
    #50 rst = 0; 
    #20 newd_s = 1;
end

 initial begin
		clk = 0;
		//scl = clk;
    	forever begin
			#10 clk =  ~clk;
          	scl = sclt ? 1'b1 : clk;
		end		
 end
	
initial begin
   #90 sclt = 1; sda_i = 1;
   //#20 sclt = 1; sda_i = 0;
   #20 sclt = 0; sda_i = 0;
   for(j=0; j<8; j=j+1)
        begin
            #20 sclt = 0;
   	        sda_i = addressToSend[j];
        end
   #20 sclt = 0; //sda_t = 0;
   for(k=0; k<8; k=k+1)
        begin
            #20 sclt = 0;//sda_t = 1;
   	        sda_i = dataToSend[k];
        end
   #20 sclt = 0; //sda_t = 0;
   #20 sclt = 1; sda_i = 0;//sda_t = 1;
   #20 sclt = 0; sda_i = 1;
   #20 sclt = 0; 
         
end
endmodule

```

### Simulation Results
---
![image](https://github.com/abhinavprakash199/Protocols-/assets/120498080/f50da5ec-8727-4521-a7e8-2f54a2c90960)

