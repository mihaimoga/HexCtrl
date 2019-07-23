## **Hex Control for MFC Applications**
![](docs/img/hexctrl_mainwnd.jpg)
### Table of Contents
* [Introduction](#introduction)
* [Implementation](#implementation)
* [Installing and Using the Control](#installing-and-using-the-control)
* [Control Creation](#control-creation)
  * [Classic Approach](#classic-approach)
  * [In Dialog](#in-dialog)
* [Set the Data](#set-the-data)
* [Control Data Modes](#control-data-modes)
  * [Memory Data](#memory-data)
  * [Message Window](#message-window)
  * [Virtual Handler](#virtual-handler)
* [OnDestroy](#ondestroy)
* [Scroll Bars](#scroll-bars)
* [Methods](#methods) <details><summary>_Expand_</summary>
  * [Create](#create)
  * [CreateDialogCtrl](#createdialogctrl)
  * [SetData](#setdata)
  * [ClearData](#cleardata)
  * [SetEditMode](#seteditmode)
  * [ShowOffset](#showoffset)
  * [SetFont](#setfont)
  * [SetFontSize](#setfontsize)
  * [SetColor](#setcolor)
  * [SetCapacity](#setcapacity)
  * [IsCreated](#iscreated)
  * [IsDataSet](#isdataset)
  * [IsMutable](#ismutable)
  * [GetFontSize](#getfontsize)
  * [GetSelection](#getselection)
  * [GetMenuHandle](#GetMenuHandle)
  * [Destroy](#destroy)
   </details>
* [Structures](#structures) <details><summary>_Expand_</summary>
  * [HEXCREATESTRUCT](#hexcreatestruct)
  * [HEXCOLORSTRUCT](#hexcolorstruct)
  * [HEXDATASTRUCT](#hexdatastruct)
  * [HEXMODIFYSTRUCT](#hexmodifystruct)
  * [HEXNOTIFYSTRUCT](#hexnotifystruct)
  * [EHexDataMode](#ehexdatamode)
  </details>
* [Positioning and Sizing](#positioning-and-sizing)
* [Appearance](#appearance)
* [Licensing](#licensing)
* [Help Point](#help-point)

## [](#)Introduction
Being good low level wrapper library for Windows API in general, **MFC** was always lacking a good native controls support.
This forced people to implement their own common stuff for everyday needs.  
This **HexControl** is an attempt to expand standard **MFC** functionality, because at the moment **MFC** doesn't have native support for such feature.

## [](#)Implementation
This **HexControl** is implemented as a pure abstract virtual class, and can be used as a *child* or *float* window in any place
of your existing application. It was build and tested in Visual Studio 2019, under Windows 10.

## [](#)Installing and Using the Control
The usage of the control is quite simple:
1. Copy **HexCtrl** folder into your project's folder.
2. Add all files from **HexCtrl** folder into your project.
3. Add `#include "HexCtrl/HexCtrl.h"` where you suppose to use the control.
4. Declare `IHexCtrlPtr` member variable: `IHexCtrlPtr myHex { CreateHexCtrl() };`
5. Call `myHex->Create` method to create control instance.
6. Call `myHex->SetData` method to set the actual data to display as hex.

`IHexCtrlPtr` is, in fact, a pointer to a `IHexCtrl` pure abstract base class, wrapped either in `std::unique_ptr` or `std::shared_ptr`. You can choose whatever is best for your needs by comment/uncomment one of these alliases in `HexCtrl.h`:
```cpp
//using IHexCtrlPtr = IHexCtrlUnPtr;
using IHexCtrlPtr = IHexCtrlShPtr;
```
This wrapper is used mainly for convenience, so you don't have to bother about object lifetime, it will be destroyed automatically.
That's why there is a call to the factory function `CreateHexCtrl()` - to properly initialize a pointer.<br>

**HexCtrl** also uses its own namespace - `HEXCTRL`. So it's up to you, whether to use namespace prefix before declarations: 
```cpp
HEXCTRL::
```
or to define namespace in the source file's beginning:
```cpp
using namespace HEXCTRL;
```

## [](#)Control Creation

### [](#)Classic Approach
The `IHexCtrl::Create` method is the first method you call, it takes [`HEXCREATESTRUCT`](#hexcreatestruct) as its argument.  
You can choose whether control will behave as a *child* or independent *floating* window, by setting `fFloat` member of this struct.
```cpp
HEXCREATESTRUCT hcs;
hcs.fFloat = true;
hcs.hwndParent = m_hWnd;
bool IHexCtrl::Create(hcs);
```
`stColor` member of [`HEXCREATESTRUCT`](#hexcreatestruct) has a type of [`HEXCOLORSTRUCT`](#hexcolorstruct). This structure describes all the **HexCtrl**'s colors, and is also used in [`IHexCtrl::SetColor`](#setcolor) method.

### [](#)In Dialog
To use **HexCtrl** within `Dialog` you can, of course, create it with the [Classic Approach](#classic-approach): 
call `IHexCtrl::Create` method and provide all the necessary information.

But there is another option you can use:
1. Put **Custom Control** control from the **Toolbox** in **Visual Studio** dialog designer into your dialog template and make it desirable size.  
![](docs/img/hexctrl_vstoolbox.jpg) ![](docs/img/hexctrl_vscustomctrl.jpg)
2. Then go to the **Properties** of that control, and in the **Class** field, within the **Misc** section, type **_HexCtrl_**.  
Give the control appropriate 
**ID** of your choise (`IDC_MY_HEX` in this example). Also, here you can set the control's **Dynamic Layout** properties, so that control behaves appropriately when dialog is being resized.
![](docs/img/hexctrl_vsproperties.jpg)
3. Declare `IHexCtrlPtr` member varable within your dialog class: `IHexCtrlPtr m_myHex { CreateHexCtrl() };`
4. Call [`IHexCtrl::CreateDialogCtrl`](#createdialogctrl ) method from your dialog's `OnInitDialog` method.
```cpp
BOOL CMyDialog::OnInitDialog()
{
    CDialogEx::OnInitDialog();
    
    m_myHex->CreateDialogCtrl(IDC_MY_HEX, m_hWnd);
}
```

## [](#)Set the Data
To set a data to display in the **HexControl** use [`IHexCtrl::SetData`](#setdata) method.
The code below shows how to construct `IHexCtrlPtr` object and display first `0x1FF` bytes of the current app's memory:
```cpp
IHexCtrlPtr myHex { CreateHexCtrl() };

HEXCREATESTRUCT hcs;
hcs.hwndParent = m_hWnd;
hcs.rect = CRect(0, 0, 500, 300); //Control's rect.

myHex->Create(hcs);

HEXDATASTRUCT hds;
hds.pData = (unsigned char*)GetModuleHandle(0);
hds.ullDataSize = 0x1FF;

myHex->SetData(hds);
```
The next example displays `std::string`'s text as hex:
```cpp
std::string str = "My string";
HEXDATASTRUCT hds;
hds.pData = (unsigned char*)str.data();
hds.ullDataSize = str.size();

myHex->SetData(hds);
```

## [](#)Control Data Modes
Besides the standard classical mode, when **HexControl** just holds a pointer to some array of bytes in memory, it also has additional advanced modes it can be running in.  
Theese modes can be quite useful for instance in cases where you need to display a very large amount of data that can't fit in memory all at once.

These modes are ruled over through the [`EHexDataMode::enMode`](#ehexdatamode) member of [`HEXDATASTRUCT`](#hexdatastruct).

### [](#)Memory Data
It's the default data mode control works in.  
The [`EHexDataMode::enMode`](#ehexdatamode) of the [`HEXDATASTRUCT`](#hexdatastruct) is set to `DATA_MEMORY`, and `pData` just points to some bytes in memory.

### [](#)Message Window
If [`EHexDataMode::enMode`](#ehexdatamode) of [`HEXDATASTRUCT`](#hexdatastruct) is set to `DATA_MSG`, the control works in so called **Message Window mode**.

What it means is that when control is about to display next byte, it will first ask for this byte from the [`HEXDATASTRUCT::hwndMsg`](#hexdatastruct) window, in the form of [`WM_NOTIFY`](https://docs.microsoft.com/en-us/windows/win32/controls/wm-notify) message. This is pretty much the same as the standard **MFC List Control** works when created with `LVS_OWNERDATA` flag.  
By default the [`HEXDATASTRUCT::hwndMsg`](#hexdatastruct) is equal to the control's parent window.

To properly handle this mode you must process `WM_NOTIFY` messages in `hwndMsg` window as follows:
```cpp
BOOL CMyWnd::OnNotify(WPARAM wParam, LPARAM lParam, LRESULT* pResult)
{
    PHEXNOTIFYSTRUCT pHexNtfy = (PHEXNOTIFYSTRUCT)lParam;
    if (pHexNtfy->hdr.idFrom == IDC_MY_HEX)
    {
        switch (pHexNtfy->hdr.code)
        {
        case HEXCTRL_MSG_GETDATA:
            pHexNtfy->pData =  /*Code to set the pointer to an actual data*/;
            break;
        }
   }
}
```
`lParam` will hold a pointer to the [`HEXNOTIFYSTRUCT`](#hexnotifystruct) structure.

The first member of this structure is a standard Windows [`NMHDR`](https://docs.microsoft.com/en-us/windows/win32/api/richedit/ns-richedit-_nmhdr) struct. It will have its `UINT code` member equal to `HEXCTRL_MSG_GETDATA`, indicating that **HexControl**'s byte request has arrived.  
The `ullIndex` member of the structure is an index of the byte to be displayed. And the `pData` is the pointer to an actual byte that you have to set in response.

### [](#)Virtual Handler
If [`EHexDataMode::enMode`](#ehexdatamode) member of [`HEXDATASTRUCT`](#hexdatastruct) is set to `DATA_VIRTUAL` then all the data routine will be done through `HEXDATASTRUCT::pHexVirtual` pointer.

This pointer is of `IHexVirtual` class type, which is a pure abstract base class.
You have to derive your own class from it and implement all its public methods:
```cpp
class IHexVirtual
{
public:
    virtual ~IHexVirtual() = default;
    virtual BYTE GetByte(ULONGLONG ullIndex) = 0;            //Gets the byte data by index.
    virtual	void ModifyData(const HEXMODIFYSTRUCT& hmd) = 0; //Main routine to modify data, in fMutable=true mode.
    virtual void Undo() = 0;                                 //Undo command, through menu or hotkey.
    virtual void Redo() = 0;                                 //Redo command, through menu or hotkey.
};
```
Then provide a pointer to created object of this derived class prior to call to [`SetData`](#setdata) method in form of `HEXDATASTRUCT::pHexVirtual = &yourDerivedObject`.

## [](#)OnDestroy
When **HexControl** window, floating or child, is being destroyed it sends [`WM_NOTIFY`](https://docs.microsoft.com/en-us/windows/win32/controls/wm-notify) message to its parent window with `NMHDR::code` equal to `HEXCTRL_MSG_DESTROY`. 
So, it basically indicates to its parent that the user clicked close button, or closed window in some other way.

## [](#)Scroll Bars
When I started to work with very big data files i immediately faced one very nasty inconvenience.

The standard Windows scrollbars can only hold `signed integer` values, which is too little to scroll through many gigabytes of data. It could be some workarounds and crutches involved to overcome this, but frankly saying i'm not a big fan of this kind of approach.

That's why **HexControl** uses its own scrollbars. They work with `unsigned long long` values, which is way bigger than standard `signed ints`. 
These scrollbars behave as normal Windows scrollbars, and even reside in the non client area as the latter do.

## [](#)Methods
**HexControl** has plenty of methods that you can use to customize its appearance, and to manage its behaviour.

### [](#)Create
**`bool Create(const HEXCREATESTRUCT& hcs)`**<br>
Main initialization method.<br>
It takes `HEXCREATESTRUCT`, that you fill first, as argument.
Returns `true` if created successfully, `false` otherwise.

### [](#)CreateDialogCtrl
**`bool CreateDialogCtrl(UINT uCtrlID, HWND hwndDlg)`**<br>
Creates **HexCtrl** from custom control dialog template. Takes control id and dialog window handle as arguments.  
See [Control Creation](#in-dialog) section for more info.

### [](#)SetData
**`void SetData(const HEXDATASTRUCT& hds)`**<br>
Main method to set data to display in read-only or edit modes. Takes [`HEXDATASTRUCT`](#hexdatastruct) as an  argument.

### [](#)ClearData
**`void ClearData()`**<br>
Clears data from the **HexCtrl** view, not touching data itself.

### [](#)SetEditMode
**`void SetEditMode(bool fEnable)`**<br>
Enables or disables edit mode. In edit mode data can be modified.

### [](#)ShowOffset
**`void ShowOffset(ULONGLONG ullOffset, ULONGLONG ullSize = 1)`**<br>
Sets cursor to the `ullOffset` and selects `ullSize` bytes.

### [](#)SetFont
**`void SetFont(const LOGFONTW* pLogFontNew)`**<br>
Sets a new font for the **HexCtrl**. This font has to be monospaced.

### [](#)SetFontSize
**`void SetFontSize(UINT uiSize)`**<br>
Sets a new font size to the **HexCtrl**.

### [](#)SetColor
**`void SetColor(const HEXCOLORSTRUCT& clr)`**<br>
Sets all the colors for the control. Takes [`HEXCOLORSTRUCT`](#hexcolorstruct) as the argument.

### [](#)SetCapacity
**`void SetCapacity(DWORD dwCapacity)`**<br>
Sets the **HexCtrl** capacity.

### [](#)IsCreated
**`bool IsCreated()const`**<br>
Shows whether **HexCtrl** is created or not yet.

### [](#)IsDataSet
**`bool IsDataSet()const`**<br>
Shows whether a data was set to **HexCtrl** or not

### [](#)IsMutable
**`bool IsMutable()const`**<br>
Shows whether **HexCtrl** is currently in edit mode or not.

### [](#)GetFontSize
**`long GetFontSize()const`**<br>
Returns current font size.

### [](#)GetSelection
**`void GetSelection(ULONGLONG& ullOffset, ULONGLONG& ullSize)const`**<br>
Returns current start position (offset) of the selection as `ullOffset`, and its size as `ullSize`.

### [](#)GetMenuHandle
**`HMENU GetMenuHandle()const`**<br>
`GetMenuHandle` method retrives the `HMENU` handle of the control's context menu. You can use this handle to customize menu for your needs.

Control's internal menu uses menu `ID`s in range starting from `0x8001`. So if you wish to add your own new menu, assign menu `ID` starting from `0x9000` to not interfere.<br>
When user clicks custom menu control sends `WM_NOTIFY` message to its parent window with `LPARAM` pointing to `HEXNOTIFYSTRUCT` with its `hdr.code` member set to `HEXCTRL_MSG_MENUCLICK`. `uMenuId` field of the [`HEXNOTIFYSTRUCT`](#hexnotifystruct) will be holding `ID` of the menu clicked.

### [](#)Destroy
**`void Destroy()`**<br>
Destroys the control.<br>
You only invoke this method if you use a raw `IHexCtrl` pointer obtained by the call to `CreateRawHexCtrl` function. Otherwise don't use it.

**Remarks**<br>
You usually don't need to call this method unless you use **HexCtrl** through the raw pointer obtained by `CreateRawHexCtrl` factory function.<br>
If you use **HexCtrl** in standard way through the `IHexCtrlPtr` pointer, obtained by `CreateHexCtrl` function, this method will be called automatically.

## [](#)Structures
Below are listed all **HexCtrl** structures.

### [](#)HEXCREATESTRUCT
The main initialization struct used for control creation.
```cpp
struct HEXCREATESTRUCT
{
    HEXCOLORSTRUCT  stColor { };           //All the control's colors.
    HWND            hwndParent { };        //Parent window pointer.
    const LOGFONTW* pLogFont { };          //Font to be used, nullptr for default. This font has to be monospaced.
    RECT            rect { };              //Initial rect. If null, the window is screen centered.
    UINT            uId { };               //Control Id.
    DWORD           dwStyle { };           //Window styles, 0 for default.
    DWORD           dwExStyle { };         //Extended window styles, 0 for default.
    bool            fFloat { false };      //Is float or child (incorporated into another window)?
    bool            fCustomCtrl { false }; //It's a custom dialog control.
};
```

### [](#)HEXCOLORSTRUCT
This structure describes all control's colors. All theese colors have their default values.
```cpp
struct HEXCOLORSTRUCT
{
    COLORREF clrTextHex { GetSysColor(COLOR_WINDOWTEXT) };         //Hex chunks text color.
    COLORREF clrTextAscii { GetSysColor(COLOR_WINDOWTEXT) };       //Ascii text color.
    COLORREF clrTextSelected { GetSysColor(COLOR_HIGHLIGHTTEXT) }; //Selected text color.
    COLORREF clrTextCaption { RGB(0, 0, 180) };                    //Caption text color
    COLORREF clrTextInfoRect { GetSysColor(COLOR_WINDOWTEXT) };    //Text color of the bottom "Info" rect.
    COLORREF clrTextCursor { RGB(255, 255, 255) };                 //Cursor text color.
    COLORREF clrBk { GetSysColor(COLOR_WINDOW) };                  //Background color.
    COLORREF clrBkSelected { GetSysColor(COLOR_HIGHLIGHT) };       //Background color of the selected Hex/Ascii.
    COLORREF clrBkInfoRect { GetSysColor(COLOR_BTNFACE) };         //Background color of the bottom "Info" rect.
    COLORREF clrBkCursor { RGB(0, 0, 255) };                       //Cursor background color.
    COLORREF clrBkCursorSelected { RGB(0, 0, 200) };               //Cursor background color in selection.
};
```

### [](#)HEXDATASTRUCT
Main struct to set a data to display in the control.
```cpp
struct HEXDATASTRUCT
{
    ULONGLONG    ullDataSize { };                      //Size of the data to display, in bytes.
    ULONGLONG    ullSelectionStart { };                //Set selection at this position. Works only if ullSelectionSize > 0.
    ULONGLONG    ullSelectionSize { };                 //How many bytes to set as selected.
    HWND         hwndMsg { };                          //Window to send the control messages to. Parent window is used by default.
    IHexVirtual* pHexVirtual { };                      //Pointer to IHexVirtual data class for custom data handling.
    PBYTE        pData { };                            //Pointer to the data. Not used if it's virtual control.
    HEXDATAMODE  enMode { HEXDATAMODE::DATA_MEMORY };  //Working data mode of the control.
    bool         fMutable { false };                   //Will data be mutable (editable) or just read mode.
};
```

### [](#)HEXMODIFYSTRUCT
This structure is used internally as well as in the external notifications routine, when working in [`DATA_MSG`](#hexdatamode) and [`DATA_VIRTUAL`](#hexdatamode) modes.
```cpp
struct HEXMODIFYSTRUCT
{
    ULONGLONG ullIndex { };       //Index of the starting byte to modify.
    ULONGLONG ullSize { };        //Size to be modified.
    ULONGLONG ullDataSize { };    //Size of pData.
    PBYTE     pData { };          //Pointer to a data to be set.
    bool      fWhole { true };    //Is a whole byte or just a part of it to be modified.
    bool      fHighPart { true }; //Shows whether high or low part of the byte should be modified (if the fWhole flag is false).
    bool      fRepeat { false };  //If ullDataSize < ullSize, should data be replaced only one time or ullSize/ullDataSize times.
};
```
When `ullDataSize` is equal `ullSize`, bytes from `pData` just replace corresponding bytes as is.  
If `ullDataSize` is less then `ullSize` only `ullDataSize` bytes are replaced if `fRepeat` flag is false. If `fRepeat` is true then block by block replacement takes place `ullSize / ullDataSize` times.

For example, if `ullSize` = 9, `ullDataSize` = 3 and `fRepeat` is true, bytes in memory at `ullIndex` position are `123456789` and bytes pointed to by `pData` are `345`, then, after modification, bytes at `ullIndex` will be `345345345`.

### [](#)HEXNOTIFYSTRUCT
This struct is used in notifications routine, when data is set with the [`DATA_MSG`](#ehexdatamode) flag.
```cpp
struct HEXNOTIFYSTRUCT
{
    NMHDR     hdr { };      //Standard Windows header. For hdr.code values see HEXCTRL_MSG_* messages.
    UINT_PTR  uMenuId { };  //User defined custom menu id.
    ULONGLONG ullIndex { }; //Index of the start byte to get/send.
    ULONGLONG ullSize { };  //Size of the bytes to get/send.
    PBYTE     pData { };    //Pointer to a data to get/send.
};
using PHEXNOTIFYSTRUCT = HEXNOTIFYSTRUCT *;
```

### [](#)EHexDataMode
Enum that represents the current data mode **HexCtrl** works in. Used as [`HEXDATASTRUCT`](#hexdatastruct) member in [`SetData`](#setdata) method.
```cpp
enum class EHexDataMode : DWORD
{
    DATA_MEMORY, DATA_MSG, DATA_VIRTUAL
};
```

## [](#)Positioning and Sizing
To properly resize and position your **HexControl**'s window you may have to handle `WM_SIZE` message in its parent window, in something like this way:
```cpp
void CMyWnd::OnSize(UINT nType, int cx, int cy)
{
    ...
    ::SetWindowPos(m_myHex->GetWindowHandle(), this->m_hWnd, 0, 0, cx, cy, SWP_NOACTIVATE | SWP_NOZORDER);
}
```

## [](#)Appearance
To change control's font size — **Ctrl+MouseWheel**  
To change control's capacity — **Ctrl+Shift+MouseWheel**

## [](#)Licensing
This software is available under the **"MIT License modified with The Commons Clause".**  
Briefly: It is free for any non commercial use.  
[https://github.com/jovibor/HexCtrl/blob/master/LICENSE](https://github.com/jovibor/HexCtrl/blob/master/LICENSE)

## [](#)Help Point
If you would like to help the author in further project's development you can do it in form of donation:

[![paypal](https://www.paypalobjects.com/en_US/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=M6CX4QH8FJJDL&item_name=Donation&currency_code=USD&source=url)