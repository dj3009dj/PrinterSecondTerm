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
	2.编辑AndroidSDK时，默认是增量编译，需要删除jni/obj文件夹，然后ant编译生成libtxdevicesdk.so文件夹。
	3.放到QQLite\plugin_native_lib中后，要全量编译。
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
  
  4.支持转发功能。先看普通AIO页面的转发支持：<br/>
 <pre>
  1. 长按进入Chapter1类中的BubbleOnlongClickListener内部类的onLongClick方法。这里设置menu的参数类型：
     QQCustomMenu menu = new QQCustomMenu();
	ChatItemBuilder builder = mfactory.findItemBuilder(AIOUtils.getMessage(v), ChatAdapter1.this);
	QQCustomMenuItem[] menus = builder.getMenuItem(v);
  2. 进入AIOUtils的getMesssage方法，获得文件的类型、位置等等：
     BaseHolder holder = (BaseHolder) getHolder(view);//返回FileItemBuilder中的Holder内部类。
  3. 进入ItemBuliderFactory的findItemBuilder方法。其messageType=MASSAGE_TYPE_FILE，返回FileItemBuilder：
     ChatItemBuilder builder = mfactory.findItemBuilder(AIOUtils.getMessage(v), ChatAdapter1.this);
  4. 利用FileItemBuilder的getMenuItem方法，返回QQCustomMenuItem。这里会添加"下载"/"转发"等功能：
        public final static  int	CLOUD_TYPE_UNKNOW 	= -1;	//默认
	public final static  int	CLOUD_TYPE_ONLINE	= 0;	//在线文件
	public final static  int	CLOUD_TYPE_OFFLINE	= 1;	//离线文件
	public final static  int	CLOUD_TYPE_WEIYUN	= 2;	//微云文件
	public final static  int	CLOUD_TYPE_LOCAL	= 3;	//本地文件
     其中只要不是在线文件，就可以有"转发"功能。
 5. 将生成的menu设置成BubbleContextMenu，再设置OnDismiss监听：
    bubbleContextMenu.setOnDismissListener(this);
 6. 点击menu项，进入FileItemBuilder的onMenuItemClicked方法。如点击"转发"则进入R.id.forward_file。
 7. 转发进入ForwardRecentActivity中，调用方式doOnCreate-->doOnCreate_init-->initViews-->addSmartDeviceEntry，添加智能设备。
 8. addSmartDeviceEntry方法调用以下语句，得到DeviceInfo[]相关信息：
    SmartDeviceProxyMgr devhandle = (SmartDeviceProxyMgr)app.getBusinessHandler(QQAppInterface.DEVICEPROXYMGR_HANDLER);
 9. 得到所有的DeviceInfo之后，会调用ForwardPhotoOption的getAllowedDevices方法，先筛选出部分设备：
  product.isSupportFuncMsgType(ProductInfo.SupportFuncType_Pic) && isSupportAbility(FORWARD_ABILITY_TYPE_SMARTDEVICE)
10. 先看isSupportFuncMsgType方法，就是SupportFuncType_Pic即1和supportFuncMsgType按位与的结果。
    其中supportFuncMsgType表示：设备支持的功能消息类型。
	/**
	 * 0x01表示图片，0x02表示视频，标志位组合，全部支持表示0x03
	 */
    只要supportFuncMsgType==0x01或者0x03，返回结果都是true。
11. 再看isSupportAbility方法，就是设备是否有转发到智能设备的能力，其中FORWARD_ABILITY_TYPE_SMARTDEVICE=10。(这个判断基本都满足)
12. 在addSmartDeviceEntry方法的后面，继续筛选设备：
                                 //过滤掉未确认的共享者
					if(info.isAdmin == SDKDef.DeviceBinder.Device_Binder_Share) {
						continue;
					}
					//过滤掉云设备
					if (devhandle.isCloudDevice(info.din)) {
						continue;
					}

13.针对不能转发文件，主要是在ForwardFileOption中mForwardAbilities没有执行下面语句：
    mForwardAbilities.add(FORWARD_ABILITY_TYPE_SMARTDEVICE);
    使得addSmartDeviceEntry方法无法进入

    
 </pre>
