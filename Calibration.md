- [Flow rate](#Flow-rate)
- [Pressure Advance](#Pressure-Advance)
  1. [Line method](#Line-method)
  2. [Tower method](#Tower-method)

# Flow rate
![flowrate](https://user-images.githubusercontent.com/103989404/210137579-3fd141ad-f2da-4542-a1fd-fc4b4d673908.gif)
Flow rate calibration is a two-pass process.
Steps 
  1. Choose the printer/filament/process you want to use for the test.
  2. Select `Pass 1` in the `Calibration` menu
  3. A new project with nine blocks will be generated, each with a different flow rate modifier. Slice and print the project.
  4. Check and decide which block has the smoothest top surface.
![flowrate-pass1_resize](https://user-images.githubusercontent.com/103989404/210138585-98821729-b19e-4452-a08d-697f147d36f0.jpg)
![0-5](https://user-images.githubusercontent.com/103989404/210138714-63daae9c-6778-453a-afa9-9a976d61bfd5.jpg)

  5. Update `Flow Ratio` in filament settings. Equation: _`FlowRatio_old*(100 + modifier)/100`_. In this case, my old `Flow Ratio` is `0.98` and I pick the `+5` block, so the new value should be 0.98x(100+5)/100 = 1.029. ** Remember** to save the filament profile.
  6. Do `Pass 2` calibration. It is the same procedure as `Pass 1`, but a new project with ten blocks will be created this time. The flow rate modifiers are ranged from `-9` to `0`.
  7. Repeat steps 4 and 5. In this case, my old `Flow Ratio` is `1.029` and I pick the `-6` block, so the new value should be 1.029x(100-6)/100 = 0.96726. ** Remember** to save the filament profile.  
![pass2](https://user-images.githubusercontent.com/103989404/210139072-f2fa91a6-4e3b-4d2a-81f2-c50155e1ff6d.jpg)
![-6](https://user-images.githubusercontent.com/103989404/210139131-ee224146-b242-4c1c-ac96-35ef0ca591f1.jpg)
![image](https://user-images.githubusercontent.com/103989404/210139721-919be130-fbba-4e3a-aa58-8a563e8c7792.png)

# Pressure Advance
I provide two approaches to calibrating PA. Each method has its own Pros and Cons.  
**NOTE**: *each method has two versions: DDE(Direct Drive Extruder) and Bowden(Extruder). Make sure you choose the right version for your test.*  

### Line method
The line method is quick and straightforward to test. However, its accuracy highly depends on your first layer quality. Make you turn on your bed mesh for this test.
Steps:
  1. Select the printer/filament/process you want to use for the calibration.
  2. Print and check the result. You can choose the value of the most uniform line and update your PA value in the filament setting.
  3. In this test, `0.016` looks optimal. 
![pa_line](https://user-images.githubusercontent.com/103989404/210139630-8fd189e7-aa6e-4d03-90ab-84ab0e781f81.gif)

<img width="1003" alt="Screenshot 2022-12-31 at 12 11 10 PM" src="https://user-images.githubusercontent.com/103989404/210124449-dd828da8-a7e4-46b8-9fa2-8bed5605d9f6.png">

![line_0 016](https://user-images.githubusercontent.com/103989404/210140046-dc5adf6a-42e8-48cd-950c-5e81558da967.jpg)
![image](https://user-images.githubusercontent.com/103989404/210140079-61a4aba4-ae01-4988-9f8e-2a45a90cdb7d.png)

### Tower method
The tower method takes slightly longer, but it's not affected by first-layer quality.
The PA value will be increased by 0.002 for every 1 mm increase in height in this test.
Steps:
 1. Select the printer/filament/process you want to use for the calibration.
 2. Check each corner of the print. Mark the height that gives the best overall result.
 3. In this case, I chose 8 mm. So the PA value should be `0.002x8 = 0.016`
![tower](https://user-images.githubusercontent.com/103989404/210140231-e886b98d-280a-4464-9781-c74ed9b7d44e.jpg)

![tower_measure](https://user-images.githubusercontent.com/103989404/210140232-885b549b-e3b8-46b9-a24c-5229c9182408.jpg)


***
*Credits:*  
- *Flowrate test is inspired by [SuperSlicer](https://github.com/supermerill/SuperSlicer)*  
- *PA Line method is inspired by [K-factor Calibration Pattern](https://marlinfw.org/tools/lin_advance/k-factor.html)*     
- *PA Tower method is inspired by [Klipper](https://www.klipper3d.org/Pressure_Advance.html)*

