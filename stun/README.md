# 家庭宽带Full Cone NAT网络环境下使用Classicstun协议获取UdpClient的公网IPEndPoint

# 背景
由于IPv4地址数量稀缺，家庭用户宽带普遍为NAT共享IP地址。这种情况下两台在不同区域的设备无法直接通信，如果数据全部通过服务器的话会造成服务器的负担过大，同时受到服务器带宽限制。  
因此Classicstun协议诞生。通过Classicstun协议可获取双方的IP地址以及端口号从而实现两台计算机直接通信，不经过服务器。

# 协议介绍
[Classicstun RFC3489](https://www.rfc-editor.org/rfc/pdfrfc/rfc3489.txt.pdf)，是一种基于UDP的协议，端口为3478，广泛用于UDP P2P通信的建立。  
当本地设备向服务器发送请求时，NAT设备会为本地设备打开一个端口，使外网设备可以通过打开的端口和本地设备通信（当NAT类型为全锥型时）
它由20字节的头部，以及可选的参数组成。

目前可用的Classicstun服务器有：  
stun.syncthing.net  
stun.hot-chilli.net  
stun.fitauto.ru  
stun.miwifi.com  
~~stun.qq.com~~  

这是Classicstun的头部以C#语法表示：
```cs
[StructLayout(LayoutKind.Sequential,Pack = 1)]
unsafe struct ClassicstunHeader
{
  public ushort MessageType; //Big endian
  public ushort MessageLength; //Big endian（不包括头部长度）
  public fixed byte ID[16];
}
```

Classicstun参数结构：
```cs
[StructLayout(LayoutKind.Sequential,Pack = 1)]
unsafe struct ClassicstunAttribute
{
  public ushort AttributeType; //Big endian
  public ushort AttributeLength; //Big endian（不包括头部长度）
  //...可变长度内容
}
```

以下是获取当前UDPClient的公网IPEndPoint的函数（仅支持Full Cone NAT）：
```cs
using System.Buffers.Binary;
using System.Net;
using System.Net.Sockets;
using System.Runtime.InteropServices;

if (GetFullConeMappedAddress(out var ep))
{
    Console.WriteLine($"Full cone! {ep}");
}

bool GetFullConeMappedAddress(out IPEndPoint? result)
{
    const string stun_server = "stun.miwifi.com";
    using (Socket client = new Socket(AddressFamily.InterNetwork, SocketType.Dgram, ProtocolType.Udp))
    {
        IPEndPoint source_address = new IPEndPoint(Dns.GetHostEntry(stun_server, AddressFamily.InterNetwork).AddressList.First(), 3478);
        IPEndPoint? changed_address = null;
        IPEndPoint? mapped_address = null;
        client.Bind(new IPEndPoint(IPAddress.Any, 0));
        {
            {
                MessageHeader request = new MessageHeader();
                request.Type = BinaryPrimitives.ReverseEndianness(MessageHeader.BINDING_REQUEST);
                request.Length = 0;
                request.TransactionID = Guid.NewGuid();
                client.SendTo(request.ToArray(), source_address);
            }
            {
                byte[] buffer = new byte[1500];
                var task = client.ReceiveFromAsync(buffer, new IPEndPoint(IPAddress.Any, 0));
                if (task.Wait(1000))
                {
                    MessageHeader response = buffer.ToStructure<MessageHeader>();
                    ushort pos = (ushort)Marshal.SizeOf<MessageHeader>();
                    while (pos < BinaryPrimitives.ReverseEndianness(response.Length))
                    {
                        MessageAttribute attr = buffer.ToStructure<MessageAttribute>(pos);
                        switch (BinaryPrimitives.ReverseEndianness(attr.Type))
                        {
                            case MessageAttribute.MAPPED_ADDRESS:
                                mapped_address = new IPEndPoint(new IPAddress(attr.IP), BinaryPrimitives.ReverseEndianness(attr.Port));
                                break;
                            case MessageAttribute.CHANGED_ADDRESS:
                                changed_address = new IPEndPoint(new IPAddress(attr.IP), BinaryPrimitives.ReverseEndianness(attr.Port));
                                break;
                        }
                        pos += sizeof(ushort) * 2;
                        pos += BinaryPrimitives.ReverseEndianness(attr.Length);
                    }
                }
                else
                {
                    throw new TimeoutException();
                }
            }
        }
        if (changed_address != null)
        {
            {
                MessageHeader request = new MessageHeader();
                request.Type = BinaryPrimitives.ReverseEndianness(MessageHeader.BINDING_REQUEST);
                request.Length = BinaryPrimitives.ReverseEndianness((ushort)8);
                request.TransactionID = Guid.NewGuid();
                MessageAttribute attr = new MessageAttribute();
                attr.Type = BinaryPrimitives.ReverseEndianness(MessageAttribute.CHANGE_REQUEST);
                attr.Length = BinaryPrimitives.ReverseEndianness((ushort)4);
                attr.ChangeRequest = MessageAttribute.ChangeFlags.ChangePort | MessageAttribute.ChangeFlags.ChangeIP;
                byte[] attrBytes = attr.ToArray();
                Array.Resize(ref attrBytes, 8);
                client.SendTo([.. request.ToArray(), .. attrBytes], source_address);
            }
            {
                byte[] buffer = new byte[1500];
                var task = client.ReceiveFromAsync(buffer, new IPEndPoint(IPAddress.Any, 0));
                if (task.Wait(1000))
                {
                    result = mapped_address;
                    return task.Result.RemoteEndPoint.Equals(changed_address);
                }
                else
                {
                    throw new TimeoutException();
                }
            }
        }
        else throw new InvalidDataException();
    }
}

[StructLayout(LayoutKind.Sequential, Pack = 1)]
struct MessageHeader
{
    public const ushort BINDING_REQUEST = 0x0001;
    public const ushort BINDING_RESPONSE = 0x0101;
    public ushort Type;
    public ushort Length;
    public Guid TransactionID;
}

[StructLayout(LayoutKind.Explicit, Pack = 1)]
public struct MessageAttribute
{
    [FieldOffset(0)]
    public ushort Type;
    [FieldOffset(2)]
    public ushort Length;

    public const ushort MAPPED_ADDRESS = 0x0001;
    public const ushort SOURCE_ADDRESS = 0x0004;
    public const ushort CHANGED_ADDRESS = 0x0005;
    [FieldOffset(4)]
    public ushort ProtocolFamily;
    [FieldOffset(6)]
    public ushort Port;
    [FieldOffset(8)]
    public uint IP;

    public const ushort CHANGE_REQUEST = 0x0003;
    [FieldOffset(7)]
    public ChangeFlags ChangeRequest;

    [Flags]
    public enum ChangeFlags : byte
    {
        ChangeIP = 0b00000100,
        ChangePort = 0b00000010
    }
}
```
2024年7月3日
