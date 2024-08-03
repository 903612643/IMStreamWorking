# IMStreamWorking 使用详解
//连接服务器<br>
A_TcpServer *tcp = [[A_TcpServer alloc] init]; <br>
[tcp loginTcpServerIP:ip地址 port:端口 Success:^{<br>
} failure:^{<br>
}];  <br>
//发送单聊信息等， <br>
NSDictionary *objc1 = @{ <br>
                            @"eventId":[NSString stringWithFormat:@"%d",SINGLECHAT_MSGID], <br>
                            @"SingleChatDto":@{ <br>
                                    @"msgTime":time, <br>
                                    @"msgContent":[objc objectForKey:@"msgContent"], <br>
                                    @"msgType":@(msgType), <br>
                                    @"msgFrom":GetUserInfo.userId, <br>
                                    @"msgTo":self.sessionId, <br>
                                    @"chatType":@"1" <br>
                                    }, <br>
                            @"msgId":[A_Util getRandomStringWithNum:32] <br>
                            }; <br>
[tcp tcpRequestAPIWithObject:objc1  Completion:^(id response, NSError *error) { <br>

}];<br>

//接收消息通知<br>
 [tcp tcpResponseAPIWithObjectCompletion:^(NSDictionary *response, NSError *error) {<br>
        [USERDEFAULT setVoiceMessageRes:YES];  //设置有语音提醒<br>
        if ([[response objectForKey:@"modelName"] isEqualToString:@"FriendApplicationRpsDto"]){ //收到加好友请求<br>
        }else if ([[response objectForKey:@"modelName"] isEqualToString:@"SingleChatRpsDto"]){  //聊天消息<br>
        }else if ([[response objectForKey:@"modelName"] isEqualToString:@"LoginOutRpsDto"]){    //被动退出登录消息<br>
            //被挤下线了，一：在线被挤掉，二：重连tcp时 返回 100064证明也被挤下线了<br>
        ...<br>
  }];<br>


# IMStreamWorking 文件夹解析<br>
IMStream 文件夹输入输出流处理，数据类型转换<br>
IMServer 即时通讯服务，连接IP地址，断开链接，主动请求回调，消息监听<br>
IMProtoAPI ProtoBuf文件请求pb文件，响应pb文件<br>
IMpbobjc  java模型生成pb的文件<br>
IMNotification 通知<br>
IMSchedule 任务调度中心api注册，请求超时，请求异步监听，收到消息监听<br>
IMManager  NSStrem类管理，NSStrem代理事件处理<br>
IMUitl     工具类，json转字典，字典转json<br>

# IMStreamWorking 文件代码详解<br>
//---------------------IMStream 文件夹-----------------------------<br>
A_DataOutputStream 输出流文件<br>
A_DataInputStream  输入流文件<br>
NSStream+NSStreamAddition  NSStream分类进行CFStreamCreatePairWithSocketToCFHost连接主机<br>

//---------------------IMServer文件夹 （*框架外部调用入口）-----------------------------<br>
IMTcpServer文件<br>
//登录IP<br>
- (void)loginTcpServerIP:(NSString*)ip port:(NSInteger)point Success:(void(^)(void))success failure:(void(^)(void))failure;<br>
//断开连接<br>
- (void)disconnect;<br>
//IM请求<br>
-(void)tcpRequestAPIWithObject:(id)objc Completion:(void(^)(id response,NSError* error))completionRespone;<br>
//IM监听<br>
-(void)tcpResponseAPIWithObjectCompletion:(void(^)(NSDictionary *response,NSError* error))completionRespone;<br>

//---------------------IMProtoAPI-----------------------------<br>
A_ResponseAPI文件，数据解析<br>
ResponseMessageProto *respone  = [ResponseMessageProto parseFromData:data error:NULL];<br>
NSDictionary* result = @{<br>
                         @"modelName":respone.modelName,<br>
                         @"respone":respone.errorNo,<br>
                         @"data":respone.result,<br>
                         @"msgId":respone.msgId,<br>
                         @"eventId":respone.eventId<br>
                         };
A_RequestAPI文件，数字压缩<br>
-(NSData*)requestProtoEventName:(NSString *)eventName modelName:(NSString*)modelName eventId:(NSString*)eventId msgId:(NSString*)msgId result:(NSDictionary*)dict<br>
{<br>
    RequestMessageProto *req = [[RequestMessageProto alloc] init];<br>
    req.eventName = eventName;<br>
    req.modelName = modelName;<br>
    req.msgId = msgId;<br>
    req.result = [A_TCPAPIUtil stringForDict:dict];<br>
    req.eventId = eventId;<br>
    NSData *data = [req data];<br>
    return data;<br>
}<br>


//---------------------IMpbobjc-----------------------------<br>
pb文件根据java文件利用protobuf库进行生成<br>
RequestProto.pbobjc      请求pb文件<br>
ResponseProto.pbobjc     响应pb文件<br>
//文件模型<br>
@property(nonatomic, readwrite, copy, null_resettable) NSString *errorNo; 错误码<br>
@property(nonatomic, readwrite, copy, null_resettable) NSString *errorInfo; 错误信息<br>
@property(nonatomic, readwrite, copy, null_resettable) NSString *modelName; 模型名，例如请求好友addfirendDto,发送消息sendMessageDto<br>
@property(nonatomic, readwrite, copy, null_resettable) NSString *result;    返回结果<br>
@property(nonatomic, readwrite, copy, null_resettable) NSString *msgId;     消息ID<br>
@property(nonatomic, readwrite, copy, null_resettable) NSString *eventId;   消息事件ID<br>


//---------------------IMSchedule-----------------------------<br>
A_SuperAPI文件 <br>
/**<br>
 *  这是一个超级类，不能被直接使用<br>
 */<br>
- (void)requestWithObject:(id)object Completion:(RequestCompletion)completion<br>
{<br>
    if (![object isKindOfClass:[NSDictionary class]]) {<br>
        return;<br>
    }<br>
    //seqNo<br>
    theSeqNo ++;<br>
    _seqNo = theSeqNo;<br>
    //注册接口<br>
    BOOL registerAPI = [[A_APISchedule instance] registerApi:(id<A_APIScheduleProtocol>)self objc:object];<br>
    if (!registerAPI)<br>
    {<br>
        return;<br>
    }<br>
    //注册请求超时<br>
    if ([(id<A_APIScheduleProtocol>)self requestTimeOutTimeInterval] > 0)<br>
    {<br>
        [[A_APISchedule instance] registerTimeoutApi:(id<A_APIScheduleProtocol>)self objc:object];<br>
    }<br>
    //保存完成块<br>
    self.completion = completion;<br>
    //数据打包<br>
    Package package = [(id<A_APIScheduleProtocol>)self packageRequestObject:object];<br>
    NSMutableData* requestData = package(object,_seqNo);<br>
    //发送<br>
    if (requestData)<br>
    {<br>
        [[A_APISchedule instance] sendData:requestData];<br>
    }<br>
    
}<br>

A_APISchedule调度中心文件<br>
//注册接口<br>
- (BOOL)registerApi:(id<A_APIScheduleProtocol>)api objc:(id)objc<br>
{<br>
    NSDictionary *objcDict = objc;<br>
    _msgId = [objcDict objectForKey:@"msgId"];<br>
    __block BOOL registSuccess = NO;<br>
    dispatch_sync(self.apiScheduleQueue, ^{<br>
        if (![api analysisReturnData])<br>
        {<br>
            registSuccess = NO;<br>
        }<br>
        //请求接口有数据返回,api协议作为value，msgId作为key放入请求表中<br>
        [self->_apiRequestMap setObject:api forKey:[objcDict objectForKey:@"msgId"]];<br>
        registSuccess = YES;<br>
    });<br>
    return registSuccess;<br>
}<br>
//请求超时<br>
- (void)registerTimeoutApi:(id<A_APIScheduleProtocol>)api objc:(id)objc;<br>
{<br>
    double delayInSeconds = [api requestTimeOutTimeInterval];<br>
    if (delayInSeconds == 0) {return;}<br>
    NSDictionary *objcDict = objc;<br>
    dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delayInSeconds * NSEC_PER_SEC));<br>
    dispatch_after(popTime, dispatch_get_main_queue(), ^(void){<br>
        if ([[self->_apiRequestMap allKeys] containsObject:[objcDict objectForKey:@"msgId"]])<br>
        {<br>
            [[A_SundriesCenter instance] pushTaskToSerialQueue:^{<br>
                RequestCompletion completion = [(A_SuperAPI*)api completion];<br>
                NSError* error = [NSError errorWithDomain:@"请求超时" code:Timeout userInfo:nil];<br>
                dispatch_sync(dispatch_get_main_queue(), ^{<br>
                    if (completion) {<br>
                        completion(nil,error);<br>
                    }<br>
                });<br>
                
            }];<br>
        }<br>
    });<br>

}<br>

//接收数据，返回是否主动请求返回的数据还是被动监听返回的数据<br>
- (void)receiveServerData:(NSData*)data forDataType:(ServerDataType)dataType<br>
{<br>
    dispatch_async(self.apiScheduleQueue, ^{<br>
        //根据key去调用注册api的completion<br>
        id<A_APIScheduleProtocol> api = self->_apiRequestMap[dataType.msgId];<br>
        if (api)<br>
        {<br>
            RequestCompletion completion = [(A_SuperAPI*)api completion];<br>
            Analysis analysis = [api analysisReturnData];<br>
            id response = analysis(data);<br>
            [self->_apiRequestMap removeObjectForKey:dataType.msgId];<br>
            dispatch_async(dispatch_get_main_queue(), ^{<br>
                @try {<br>
                    completion(response,nil);<br>
                }<br>
                @catch (NSException *exception) {<br>
                    NSLog(@"completion,response is nil");<br>
                }<br>
            });<br>
        } else<br>
        {<br>
            id<A_APIUnrequestScheduleProtocol> unrequestAPI = self->_apiResponseMap[MAP_RESPONE_KEY];<br>
            if (unrequestAPI)<br>
            {<br>
                UnrequestAPIAnalysis unrequestAnalysis = [unrequestAPI unrequestAnalysis];<br>
                id object = unrequestAnalysis(data);<br>
                ReceiveData received = [(A_UnrequestSuperAPI*)unrequestAPI receivedData];<br>
                dispatch_async(dispatch_get_main_queue(), ^{<br>
                    @try {<br>
                        received(object,nil);<br>
                    }<br>
                    @catch (NSException *exception) {<br>
                        NSLog(@"completion,response is nil");<br>
                    }<br>
                });<br>
            }<br>
            
        }<br>
    });<br>
  }<br>

    



