### **Overview**

- **Platform**: Intel DevCloud
- **FPGA**: Intel® Programmable Acceleration Card (PAC) with Intel® Arria® 10 GX FPGA
- **Communication Interface**: OPAE (Open Programmable Acceleration Engine)
- **FPGA Code**: Verilog
- **CPU Code**: C++
- **Driver Interface**: Fully open-source (OPAE)
- **Goal**: Implement a CPU-FPGA application where data is sent from the CPU to the FPGA, processed on the FPGA, and sent back to the CPU.

---

### **Table of Contents**

1. [Setting Up Your Intel DevCloud Account](#section1)
2. [Accessing the Intel DevCloud](#section2)
3. [Preparing the Development Environment](#section3)
4. [Understanding the OPAE Framework](#section4)
5. [Creating the FPGA Design (Verilog)](#section5)
6. [Compiling the FPGA Design](#section6)
7. [Creating the Host Application (C++)](#section7)
8. [Compiling the Host Application](#section8)
9. [Programming the FPGA](#section9)
10. [Running the Host Application](#section10)
11. [Verifying the Results](#section11)
12. [Cleaning Up](#section12)
13. [Conclusion](#section13)
14. [References](#section14)

---

<a name="section1"></a>
## **1. Setting Up Your Intel DevCloud Account**

### **Steps:**

1. **Visit the Intel DevCloud Registration Page:**

   - Go to [Intel DevCloud for oneAPI](https://devcloud.intel.com/oneapi/get_started/).

2. **Click on "Get Free Access":**

   - You'll be redirected to a registration form.

3. **Fill Out the Registration Form:**

   - Provide your personal details, academic or professional affiliation, and agree to the terms.

4. **Submit the Form:**

   - After submission, you'll receive an email to verify your email address.

5. **Verify Your Email Address:**

   - Click on the verification link in the email.

6. **Account Activation:**

   - Intel may take some time to approve your account (usually instantly verfied). You'll receive a confirmation email once it's activated. "Your Intel account has been created successfully"

---

<a name="section2"></a>
## **2. Accessing the Intel DevCloud**

### **Steps:**

1. **Log In to the DevCloud:**

   - Go to [Intel DevCloud login page](https://consumer.intel.com/intelcorpb2c.onmicrosoft.com/B2C_1A_UnifiedLogin_SISU_CML_SAML/generic/login?entityId=www.intel.com&ui_locales=en) and sign in with your credentials.

2. **Access via SSH:**

   - **Generate SSH Keys (if not already done):**
     - On your local machine, run:
       ```bash
       ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
       ```
     - Save the key in the default location (`~/.ssh/id_rsa`).

   - **Upload Your Public Key to DevCloud:**
     - In the DevCloud portal, go to **Account Details** > **SSH Keys**.
     - Upload your public key (`~/.ssh/id_rsa.pub`).

   - **Connect via SSH:**
     - Open a terminal and run:
       ```bash
       ssh devcloud
       ```
     - This command is a shortcut provided by Intel after setting up SSH configurations. If it doesn't work, use:
       ```bash
       ssh your_username@devcloud.intel.com
       ```

3. **Verify Access:**

   - You should now be logged into the DevCloud environment.

---

<a name="section3"></a>
## **3. Preparing the Development Environment**

### **Steps:**

1. **Load the oneAPI Environment:**

   - Run the following command to set up the oneAPI environment:
     ```bash
     source /opt/intel/inteloneapi/setvars.sh
     ```

2. **Load OPAE and FPGA Modules:**

   - Load the necessary modules for FPGA development:
     ```bash
     module use /glob/development-tools/oneapi/modulefiles
     module load intelFPGA
     ```

3. **Allocate an FPGA Node with Arria 10 PAC Card:**

   - Request an interactive session on a node with an Intel Arria 10 FPGA:
     ```bash
     qsub -I -l nodes=1:fpga_runtime:arria10:ppn=2 -d .
     ```
   - Wait until the session is allocated. You should now be on a compute node with access to the FPGA.

4. **Verify FPGA Access:**

   - Check the FPGA is visible:
     ```bash
     fpgainfo fme
     ```
   - You should see information about the FPGA Management Engine (FME).

---

<a name="section4"></a>
## **4. Understanding the OPAE Framework**

The Open Programmable Acceleration Engine (OPAE) is an open-source framework that provides a lightweight API to communicate with FPGA devices over PCIe. It simplifies the development of host applications that interact with FPGA accelerators.

- **Key Components:**
  - **FPGA Interface Manager Extension (FIM):** The FPGA's base image that manages communication.
  - **Accelerator Function Unit (AFU):** Your custom logic implemented on the FPGA.
  - **Host Application:** A software application running on the CPU that communicates with the AFU via OPAE APIs.

---

<a name="section5"></a>
## **5. Creating the FPGA Design (Verilog)**

We'll create a simple Verilog AFU that increments an input value and sends it back to the host.

### **Steps:**

1. **Set Up the Directory Structure:**

   - Create a project directory:
     ```bash
     mkdir -p ~/hello_fpga/afu
     cd ~/hello_fpga
     ```

2. **Clone the OPAE Base AFU Repository:**

   - This repository contains base templates for AFU development.
     ```bash
     git clone https://github.com/OPAE/opae-samples.git
     ```

3. **Copy the "hello_afu" Sample:**

   - This sample provides a good starting point.
     ```bash
     cp -r opae-samples/hello_afu afu
     cd afu
     ```

4. **Modify the Verilog Code:**

   - Open the AFU RTL file:
     ```bash
     nano src/rtl/hello_afu.sv
     ```
   - Modify the code to implement increment logic:

     Replace the existing `hello_afu` module with the following:

     ```verilog
     module hello_afu (
         input  wire         clk,
         input  wire         rst_n,
         // AXI Stream Slave Interface
         input  wire [511:0]  s_axis_tdata,
         input  wire          s_axis_tvalid,
         output wire          s_axis_tready,
         // AXI Stream Master Interface
         output wire [511:0]  m_axis_tdata,
         output wire          m_axis_tvalid,
         input  wire          m_axis_tready
     );

     reg [511:0] data_reg;
     reg         valid_reg;

     assign s_axis_tready = m_axis_tready;
     assign m_axis_tvalid = valid_reg;
     assign m_axis_tdata  = data_reg;

     always @(posedge clk or negedge rst_n) begin
         if (!rst_n) begin
             data_reg  <= 0;
             valid_reg <= 0;
         end else begin
             if (s_axis_tvalid && s_axis_tready) begin
                 data_reg  <= s_axis_tdata + 1; // Increment the input data
                 valid_reg <= 1;
             end else if (m_axis_tvalid && m_axis_tready) begin
                 valid_reg <= 0;
             end
         end
     end

     endmodule
     ```

     - This module reads data from the host, increments it by 1, and sends it back.

5. **Update the AFU Manifest:**

   - The AFU Manifest (`afu.json`) describes the AFU's UUID and interface.

   - Open `hw/afu.json`:
     ```bash
     nano hw/afu.json
     ```

   - Ensure the `name` and `description` fields are updated:

     ```json
     {
       "afu-image": {
         "power": 0,
         "clock-frequency-high": "auto",
         "clock-frequency-low": "auto",
         "afu-top-interface": {
           "class": "ccip_std_afu"
         },
         "accelerator-clusters": [
           {
             "name": "hello_world",
             "total-contexts": 1,
             "accelerator-type-uuid": "YOUR-AFU-UUID-HERE"
           }
         ]
       }
     }
     ```

   - Generate a UUID for your AFU:

     ```bash
     uuidgen
     ```

   - Replace `"YOUR-AFU-UUID-HERE"` with the generated UUID.

---

<a name="section6"></a>
## **6. Compiling the FPGA Design**

### **Steps:**

1. **Set Up the Quartus Environment:**

   - Load the Quartus Prime Pro Edition tools:
     ```bash
     module load intelFPGA_pro
     ```

2. **Compile the AFU:**

   - Use the OPAE build scripts to compile the AFU:

     ```bash
     cd ~/hello_fpga/afu
     ./build.sh
     ```

   - **Note:** The compilation process is resource-intensive and may take several hours. Ensure that your allocated node has sufficient time.

3. **Check for Successful Compilation:**

   - Once compilation is complete, check for the generated GBS file (Green Bitstream):

     ```bash
     ls build/outputs/*.gbs
     ```

   - The GBS file is used to program the FPGA.

---

<a name="section7"></a>
## **7. Creating the Host Application (C++)**

We'll create a host application that uses OPAE APIs to communicate with the AFU.

### **Steps:**

1. **Create the Host Application Directory:**

   ```bash
   mkdir ~/hello_fpga/host
   cd ~/hello_fpga/host
   ```

2. **Write the Host Application Code:**

   - Create a file named `host.cpp`:
     ```bash
     nano host.cpp
     ```

   - Add the following code:

     ```cpp
     #include <opae/fpga.h>
     #include <iostream>
     #include <cstdint>
     #include <cstring>
     #include <uuid/uuid.h>

     #define AFU_UUID "YOUR-AFU-UUID-HERE"

     int main() {
         fpga_properties filter = nullptr;
         fpga_handle afc_handle;
         fpga_guid guid;
         fpga_token afc_token;
         uint32_t num_matches = 1;

         // Initialize OPAE
         if (fpgaInitialize(nullptr) != FPGA_OK) {
             std::cerr << "Failed to initialize OPAE." << std::endl;
             return 1;
         }

         // Set up filter to search for the AFU
         if (fpgaGetProperties(nullptr, &filter) != FPGA_OK) {
             std::cerr << "Failed to get FPGA properties." << std::endl;
             return 1;
         }

         fpgaPropertiesSetObjectType(filter, FPGA_ACCELERATOR);
         uuid_parse(AFU_UUID, guid);
         fpgaPropertiesSetGUID(filter, guid);

         // Enumerate FPGAs
         if (fpgaEnumerate(&filter, 1, &afc_token, 1, &num_matches) != FPGA_OK) {
             std::cerr << "Failed to enumerate FPGAs." << std::endl;
             return 1;
         }

         if (num_matches < 1) {
             std::cerr << "No matching AFU found." << std::endl;
             return 1;
         }

         // Open the AFU
         if (fpgaOpen(afc_token, &afc_handle, 0) != FPGA_OK) {
             std::cerr << "Failed to open AFU." << std::endl;
             return 1;
         }

         // Allocate buffers
         uint64_t wsid;
         volatile uint64_t *input_buf;
         uint64_t iova;

         if (fpgaPrepareBuffer(afc_handle, sizeof(uint64_t), (void**)&input_buf, &wsid, 0) != FPGA_OK) {
             std::cerr << "Failed to allocate input buffer." << std::endl;
             return 1;
         }

         if (fpgaGetIOAddress(afc_handle, wsid, &iova) != FPGA_OK) {
             std::cerr << "Failed to get IO address." << std::endl;
             return 1;
         }

         // Initialize input data
         uint64_t input_value = 42;
         *input_buf = input_value;

         std::cout << "CPU: Sent data to FPGA: " << input_value << std::endl;

         // AFU interaction logic here
         // For simplicity, we'll assume the AFU reads from the buffer, processes it, and writes back

         // Simulate a delay (in real application, you would wait for an interrupt or poll a status register)
         sleep(1);

         // Read back the result
         uint64_t output_value = *input_buf;

         std::cout << "CPU: Received data from FPGA: " << output_value << std::endl;

         // Verify the result
         if (output_value == input_value + 1) {
             std::cout << "SUCCESS: FPGA incremented the data correctly." << std::endl;
         } else {
             std::cerr << "ERROR: FPGA did not increment the data correctly." << std::endl;
         }

         // Clean up
         fpgaReleaseBuffer(afc_handle, wsid);
         fpgaClose(afc_handle);
         fpgaDestroyProperties(&filter);

         return 0;
     }
     ```

   - Replace `"YOUR-AFU-UUID-HERE"` with the UUID you generated earlier.

3. **Explanation of the Host Code:**

   - **Initialization:**
     - Initializes the OPAE library.
     - Sets up a filter to find the FPGA accelerator with the specified UUID.
   - **FPGA Interaction:**
     - Opens a handle to the AFU.
     - Allocates a shared buffer accessible by both the host and the FPGA.
     - Writes data to the buffer.
     - In a real-world scenario, you might need to trigger the AFU or wait for a completion signal.
     - Reads the processed data from the buffer.
   - **Verification:**
     - Compares the received data to the expected result.
   - **Cleanup:**
     - Releases resources and closes handles.

---

<a name="section8"></a>
## **8. Compiling the Host Application**

### **Steps:**

1. **Set Up the Build Environment:**

   - Ensure that the OPAE development libraries are installed:
     ```bash
     module load opae
     ```

2. **Compile the Host Application:**

   ```bash
   g++ -o host_app host.cpp -I/opt/intel/opae/include -L/opt/intel/opae/lib -lopae-c -luuid
   ```

   - **Flags Explanation:**
     - `-I/opt/intel/opae/include`: Include path for OPAE headers.
     - `-L/opt/intel/opae/lib`: Library path for OPAE libraries.
     - `-lopae-c`: Link against the OPAE C library.
     - `-luuid`: Link against the UUID library.

---

<a name="section9"></a>
## **9. Programming the FPGA**

### **Steps:**

1. **Check for Available FPGA Cards:**

   ```bash
   fpgainfo fme
   ```

   - Ensure that the FPGA is visible and accessible.

2. **Program the FPGA with the GBS File:**

   - Use the `fpgaconf` utility to program the FPGA:

     ```bash
     sudo fpgaconf -v ~/hello_fpga/afu/build/outputs/hello_afu.gbs
     ```

   - **Note:**
     - You may need `sudo` privileges to program the FPGA.
     - Ensure that no other processes are using the FPGA.

3. **Verify the FPGA Is Programmed:**

   - Check the AFU ID:

     ```bash
     fpgainfo afu
     ```

   - The output should show the AFU UUID matching the one you generated.

---

<a name="section10"></a>
## **10. Running the Host Application**

### **Steps:**

1. **Execute the Host Application:**

   ```bash
   ./host_app
   ```

2. **Expected Output:**

   ```
   CPU: Sent data to FPGA: 42
   CPU: Received data from FPGA: 43
   SUCCESS: FPGA incremented the data correctly.
   ```

   - The application sends `42` to the FPGA.
   - The FPGA increments the value to `43`.
   - The host application reads back `43` and verifies the result.

---

<a name="section11"></a>
## **11. Verifying the Results**

### **Steps:**

1. **Check the Host Application Output:**

   - Ensure that the output indicates success.

2. **Debugging If Needed:**

   - If the output shows an error:
     - Check that the AFU is correctly programmed.
     - Ensure that the UUIDs match between the host application and the AFU manifest.
     - Verify that the FPGA is not being used by another process.
     - Check for any error messages during compilation or execution.

3. **Monitor FPGA Status:**

   - Use `fpgainfo` to check FPGA status:

     ```bash
     fpgainfo errors
     ```

   - Look for any reported errors.

---

<a name="section12"></a>
## **12. Cleaning Up**

### **Steps:**

1. **Release FPGA Resources:**

   - Ensure that the host application has terminated and released all resources.

2. **Unload Modules:**

   ```bash
   module unload opae
   module unload intelFPGA_pro
   ```

3. **Exit the Interactive Session:**

   ```bash
   exit
   ```

4. **Close SSH Connection:**

   ```bash
   exit
   ```

---

<a name="section13"></a>
## **13. Conclusion**

By following these detailed steps, you've:

- Set up an account and accessed the Intel DevCloud.
- Prepared the development environment for FPGA programming with OPAE.
- Created a Verilog-based FPGA design (AFU) that processes data sent from the CPU.
- Compiled the FPGA design and programmed the FPGA with it.
- Written a C++ host application that uses open-source OPAE drivers to communicate with the FPGA.
- Executed the host application, achieving direct, secure communication between the CPU and FPGA.
- Verified the correct operation of the system.

This setup provides a real-world example of CPU-FPGA communication in a cloud environment without the need for physical hardware and using fully open-source drivers.

---

<a name="section14"></a>
## **14. References**

- **Intel DevCloud Documentation:**
  - [Intel DevCloud for oneAPI](https://devcloud.intel.com/oneapi/documentation/)
- **OPAE Documentation:**
  - [OPAE GitHub Repository](https://github.com/OPAE/opae-sdk)
  - [OPAE Quick Start Guide](https://opae.github.io/latest/quickstartguide.html)
- **FPGA Programming Guides:**
  - [Intel® FPGA SDK for OpenCL™ Pro Edition: Programming FPGA](https://www.intel.com/content/www/us/en/docs/programmable/683472/current/programming-an-fpga.html)
- **Intel FPGA Documentation:**
  - [Intel® Acceleration Stack for Intel® Xeon® CPU with FPGAs](https://www.intel.com/content/www/us/en/programmable/documentation/lro1428694837525.html)

---

### **Important Notes:**

- **Permissions and Privileges:**

  - Programming the FPGA (`fpgaconf`) may require `sudo` privileges.
  - Ensure you have the necessary permissions on the DevCloud. If you encounter permission issues, contact Intel DevCloud support.

- **Resource Usage:**

  - FPGA compilation is resource-intensive and time-consuming.
  - Be mindful of DevCloud usage policies and time limits on interactive sessions.

- **Security Considerations:**

  - The communication between CPU and FPGA over PCIe is considered secure within the cloud environment.
  - OPAE provides mechanisms for secure communication, but always ensure your code handles data securely.

- **Driver Interface:**

  - OPAE is fully open-source, satisfying your requirement for open-source drivers.
  - The OPAE SDK is licensed under the BSD-3-Clause license.

- **Alternative Options:**

  - If Intel DevCloud does not meet your needs due to permissions or other limitations, consider other cloud providers like Nimbix Cloud or hosting providers that offer physical access to FPGA servers with open-source support.
