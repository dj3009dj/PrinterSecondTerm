# PrinterSecondTerm
1.用户进入AIO页面，统计上报，提醒HP增强轮询频率。物联的AIO页面在DeviceMsgChatPie中：<br/>
<pre>
			SmartDeviceProxyMgr mgr = (SmartDeviceProxyMgr)app.getBusinessHandler(QQAppInterface.DEVICEPROXYMGR_HANDLER);
			isCloudPrinter = mgr.isCloudDevice(din) && mgr.isSpecialDeviceType(din, SDKDef.DeviceType.PRINTER);
			if(isCloudPrinter){
				mgr.sendCloudPrintCmd(din,"","","",0,1,1);
			}
	</pre>
	
  app的类型是主进程的QQAppInterface，在物联进程拿不到；这里的目的是判断是否为云打印机，如果是通过以下步骤：<br/>
	<pre>
	1.调用SmartDeviceProxyMgr的sendCloudPrintCmd方法。该方法通过SmartDeviceIPCHost的postRemoteNotify方法推送远程通知。
	2.调用AIDL中的ipc文件中的ISmartDeviceService接口的transferAsync方法。该方法最终调用TencentIMEngine的sendCloudPrintCmd方法。
	3.该JNI方法调用,CloudDeviceMgr::SendCloudPrintCmd方法，这里做了判断。如果fileType=0：
	        int type1= CommonCSProto::ReqBody::DeviceAccessNoticeReq::uint64_din;
        int type2= CommonCSProto::ReqBody::DeviceAccessNoticeReq::uint32_pid;
        printReq.AddUInt64(CommonCSProto::ReqBody::DeviceAccessNoticeReq::uint64_din, u64Din);
        printReq.AddUInt32(CommonCSProto::ReqBody::DeviceAccessNoticeReq::uint32_pid, uProductId);
  4.SendCloudPrintCmd方法回调OnCSCallback方法，OnCSCallback方法调用OnRecvCSReply方法，OnRecvCSReply方法回调DeviceMgr::OnPrintJobNotify方法。
  5.从.so文件回到TencentIMEngine.OnPrintJobNotify方法，其中Bundle[{strJobId=, nType=4, nEventValue=0, uDin=0}]，该方法会发送广播。
  6.DeviceFileHandler类的DeviceNotifyReceiver会接收广播。由于strJobId为空：导致以下语句crash:
            Session session = mServiceSessionMap.get(Long.parseLong(strJobId));
	</pre>
	这里是不改变后台代码的运行结果：<br/>
	<pre>
	1.TencentIMEngine.OnPrintJobNotify方法返回：Bundle[{strJobId=, nType=1, uDin=800331}]
	</pre>

2.修改打印机状态，使其不显示"在线"和"离线"状态。具体步骤如下：<br/>
  <pre>
  1.联系人列表的操作在Contacts中，其中有一项专门操作列表的适配器：
     BuddyListAdapter mGroupingBuddyListAdapter;
  2.BuddyListAdapter中的getBuddyChildView方法用来显示"我的设备"/"我的电脑"...，其中利用以下方法判断是否为智能设备：
     AppConstants.SMARTDEVICE_UIN.equals(buddy.uin)
  3.判断完是智能设备后，对于“防丢器”这种无状态设备而言，buddy.status==11。打印机要做的与它类似，只要知道布局资源在哪就明白了：
     convertView = LayoutInflater.from(mContext).inflate(R.layout.contact_list_item_for_buddy, parent, false);
  4.进入LightAppActivity之后，也需要去除状态。首先调用DevicePresenterImpl的init方法。
  5.进入init方法后，调用setDevStatus方法。setDevStatus方法调用LightAppActivity的setOnlineStatus方法。这里判断是否为打印机：
    mDeviceView.setOnlineStatus(DEVICE_STATUS_HIDE_VIEW);//隐藏状态
  </pre>
  
  3.增加打印异常错误码。具体步骤如下：<br/>
  <pre>
  1.JNI文件中设置nType:
    enum PrintJobNotify
    {
        PrintJobNotify_Start        = 1,    // 云打印作业启动
        PrintJobNotify_Progress     = 2,    // 云打印作业进度
        PrintJobNotify_Status       = 3,    // 云打印状态，数字表示。错误码：1代表缺墨，2代表缺纸，3其他。
        PrintMachineNotify_Status       = 4,    // 云打印机状态，数字表示。错误码：1代表缺墨，2代表缺纸，3其他。
    };
  2.TencentIMEngine的OnPrintJobNotufy方法返回nType，依然会返回到DeviceFileHandler类的DeviceNotifyReceiver中接收广播。
  3.其中当nType==3时，设置mPrinterJobErrorMap：
        case SDKDef.CloudPrintJobNotify.CPJob_Status:
							//打印的状态，数字表示。错误码：0其他， 1代表缺纸，2代表缺墨，3代表离线
							mPrinterJobErrorMap.put(session.uSessionID, nEventValue);
	4.设置后进入到DeviceComnFileMsgProcessor类的onServiceSessionCompletef方法，该方法获取打印任务错误码：
    	if (mPrinterJobErrorMap.containsKey(sessionID)) {
	    		code = mPrinterJobErrorMap.get(sessionID);
	    		mPrinterJobErrorMap.remove(sessionID);
	    	}
	5.获取错误码后，将mr.nFileStatus设置为错误码。
	6.在DeviceFileItemBuilder类中的showFileState方法中设置错误提醒：
				case DeviceFileHandler.status_printer_noget_error:
				holder.fileStateTv.setVisibility(View.VISIBLE);
				if(!message.isSend()&&mISCloudPrint){
					holder.fileStateTv.setText(space+mContext.getString(R.string.device_msg_sendcloudprint_fail)+"(错误07)");
				}
				break;
  </pre>
