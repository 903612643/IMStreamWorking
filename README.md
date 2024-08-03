# IMStreamWorking 使用详解
//连接服务器
A_TcpServer *tcp = [[A_TcpServer alloc] init];
[tcp loginTcpServerIP:ip地址 port:端口 Success:^{
} failure:^{
}];
//发送单聊信息等，
NSDictionary *objc1 = @{
                            @"eventId":[NSString stringWithFormat:@"%d",SINGLECHAT_MSGID],
                            @"SingleChatDto":@{
                                    @"msgTime":time,
                                    @"msgContent":[objc objectForKey:@"msgContent"],
                                    @"msgType":@(msgType),
                                    @"msgFrom":GetUserInfo.userId,
                                    @"msgTo":self.sessionId,
                                    @"chatType":@"1"
                                    },
                            @"msgId":[A_Util getRandomStringWithNum:32]
                            };
[tcp tcpRequestAPIWithObject:objc1  Completion:^(id response, NSError *error) {

}];

//接收消息通知
 [tcp tcpResponseAPIWithObjectCompletion:^(NSDictionary *response, NSError *error) {
        [USERDEFAULT setVoiceMessageRes:YES];  //设置有语音提醒
        if ([[response objectForKey:@"modelName"] isEqualToString:@"FriendApplicationRpsDto"]){ //收到加好友请求
        }else if ([[response objectForKey:@"modelName"] isEqualToString:@"SingleChatRpsDto"]){  //聊天消息
        }else if ([[response objectForKey:@"modelName"] isEqualToString:@"LoginOutRpsDto"]){    //被动退出登录消息
            //被挤下线了，一：在线被挤掉，二：重连tcp时 返回 100064证明也被挤下线了
        ...
  }];


# IMStreamWorking 文件夹解析
IMStream 文件夹输入输出流处理，数据类型转换
IMServer 即时通讯服务，连接IP地址，断开链接，主动请求回调，消息监听
IMProtoAPI ProtoBuf文件请求pb文件，响应pb文件
IMpbobjc  java模型生成pb的文件
IMNotification 通知
IMSchedule 任务调度中心api注册，请求超时，请求异步监听，收到消息监听
IMManager  NSStrem类管理，NSStrem代理事件处理
IMUitl     工具类，json转字典，字典转json

# IMStreamWorking 文件代码详解
//---------------------IMStream 文件夹-----------------------------
A_DataOutputStream 输出流文件
A_DataInputStream  输入流文件
NSStream+NSStreamAddition  NSStream分类进行CFStreamCreatePairWithSocketToCFHost连接主机

//---------------------IMServer文件夹 （*框架外部调用入口）-----------------------------
IMTcpServer文件
//登录IP
- (void)loginTcpServerIP:(NSString*)ip port:(NSInteger)point Success:(void(^)(void))success failure:(void(^)(void))failure;
//断开连接
- (void)disconnect;
//IM请求
-(void)tcpRequestAPIWithObject:(id)objc Completion:(void(^)(id response,NSError* error))completionRespone;
//IM监听
-(void)tcpResponseAPIWithObjectCompletion:(void(^)(NSDictionary *response,NSError* error))completionRespone;

//---------------------IMProtoAPI-----------------------------
A_ResponseAPI文件，数据解析
ResponseMessageProto *respone  = [ResponseMessageProto parseFromData:data error:NULL];
NSDictionary* result = @{
                         @"modelName":respone.modelName,
                         @"respone":respone.errorNo,
                         @"data":respone.result,
                         @"msgId":respone.msgId,
                         @"eventId":respone.eventId
                         };
A_RequestAPI文件，数字压缩
-(NSData*)requestProtoEventName:(NSString *)eventName modelName:(NSString*)modelName eventId:(NSString*)eventId msgId:(NSString*)msgId result:(NSDictionary*)dict
{
    RequestMessageProto *req = [[RequestMessageProto alloc] init];
    req.eventName = eventName;
    req.modelName = modelName;
    req.msgId = msgId;
    req.result = [A_TCPAPIUtil stringForDict:dict];
    req.eventId = eventId;
    NSData *data = [req data];
    return data;
}


//---------------------IMpbobjc-----------------------------
pb文件根据java文件利用protobuf库进行生成
RequestProto.pbobjc      请求pb文件
ResponseProto.pbobjc     响应pb文件
//文件模型
@property(nonatomic, readwrite, copy, null_resettable) NSString *errorNo; 错误码
@property(nonatomic, readwrite, copy, null_resettable) NSString *errorInfo; 错误信息
@property(nonatomic, readwrite, copy, null_resettable) NSString *modelName; 模型名，例如请求好友addfirendDto,发送消息sendMessageDto
@property(nonatomic, readwrite, copy, null_resettable) NSString *result;    返回结果
@property(nonatomic, readwrite, copy, null_resettable) NSString *msgId;     消息ID
@property(nonatomic, readwrite, copy, null_resettable) NSString *eventId;   消息事件ID


//---------------------IMSchedule-----------------------------
A_SuperAPI文件 
/**
 *  这是一个超级类，不能被直接使用
 */
- (void)requestWithObject:(id)object Completion:(RequestCompletion)completion
{
    if (![object isKindOfClass:[NSDictionary class]]) {
        return;
    }
    //seqNo
    theSeqNo ++;
    _seqNo = theSeqNo;
    //注册接口
    BOOL registerAPI = [[A_APISchedule instance] registerApi:(id<A_APIScheduleProtocol>)self objc:object];
    if (!registerAPI)
    {
        return;
    }
    //注册请求超时
    if ([(id<A_APIScheduleProtocol>)self requestTimeOutTimeInterval] > 0)
    {
        [[A_APISchedule instance] registerTimeoutApi:(id<A_APIScheduleProtocol>)self objc:object];
    }
    //保存完成块
    self.completion = completion;
    //数据打包
    Package package = [(id<A_APIScheduleProtocol>)self packageRequestObject:object];
    NSMutableData* requestData = package(object,_seqNo);
    //发送
    if (requestData)
    {
        [[A_APISchedule instance] sendData:requestData];
    }
    
}

A_APISchedule调度中心文件
//注册接口
- (BOOL)registerApi:(id<A_APIScheduleProtocol>)api objc:(id)objc
{
    NSDictionary *objcDict = objc;
    _msgId = [objcDict objectForKey:@"msgId"];
    __block BOOL registSuccess = NO;
    dispatch_sync(self.apiScheduleQueue, ^{
        if (![api analysisReturnData])
        {
            registSuccess = NO;
        }
        //请求接口有数据返回,api协议作为value，msgId作为key放入请求表中
        [self->_apiRequestMap setObject:api forKey:[objcDict objectForKey:@"msgId"]];
        registSuccess = YES;
    });
    return registSuccess;
}
//请求超时
- (void)registerTimeoutApi:(id<A_APIScheduleProtocol>)api objc:(id)objc;
{
    double delayInSeconds = [api requestTimeOutTimeInterval];
    if (delayInSeconds == 0) {return;}
    NSDictionary *objcDict = objc;
    dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delayInSeconds * NSEC_PER_SEC));
    dispatch_after(popTime, dispatch_get_main_queue(), ^(void){
        if ([[self->_apiRequestMap allKeys] containsObject:[objcDict objectForKey:@"msgId"]])
        {
            [[A_SundriesCenter instance] pushTaskToSerialQueue:^{
                RequestCompletion completion = [(A_SuperAPI*)api completion];
                NSError* error = [NSError errorWithDomain:@"请求超时" code:Timeout userInfo:nil];
                dispatch_sync(dispatch_get_main_queue(), ^{
                    if (completion) {
                        completion(nil,error);
                    }
                });
                
            }];
        }
    });

}

//接收数据，返回是否主动请求返回的数据还是被动监听返回的数据
- (void)receiveServerData:(NSData*)data forDataType:(ServerDataType)dataType
{
    dispatch_async(self.apiScheduleQueue, ^{
        //根据key去调用注册api的completion
        id<A_APIScheduleProtocol> api = self->_apiRequestMap[dataType.msgId];
        if (api)
        {
            RequestCompletion completion = [(A_SuperAPI*)api completion];
            Analysis analysis = [api analysisReturnData];
            id response = analysis(data);
            [self->_apiRequestMap removeObjectForKey:dataType.msgId];
            dispatch_async(dispatch_get_main_queue(), ^{
                @try {
                    completion(response,nil);
                }
                @catch (NSException *exception) {
                    NSLog(@"completion,response is nil");
                }
            });
        } else
        {
            id<A_APIUnrequestScheduleProtocol> unrequestAPI = self->_apiResponseMap[MAP_RESPONE_KEY];
            if (unrequestAPI)
            {
                UnrequestAPIAnalysis unrequestAnalysis = [unrequestAPI unrequestAnalysis];
                id object = unrequestAnalysis(data);
                ReceiveData received = [(A_UnrequestSuperAPI*)unrequestAPI receivedData];
                dispatch_async(dispatch_get_main_queue(), ^{
                    @try {
                        received(object,nil);
                    }
                    @catch (NSException *exception) {
                        NSLog(@"completion,response is nil");
                    }
                });
            }
            
        }
    });
  }


    



