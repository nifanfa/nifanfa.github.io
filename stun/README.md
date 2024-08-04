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
using System.Net;
using System.Net.Sockets;

UdpClient uc = new UdpClient(0, AddressFamily.InterNetwork);
if (GetNATPublicEndPointIfFullCone(ref uc, out var mapped))
{
    Console.WriteLine($"UdpClient的公网IPEndPoint是：{mapped}");
}
else
{
    throw new Exception("NAT类型不是Full Cone");
}

unsafe bool GetNATPublicEndPointIfFullCone(ref UdpClient uc, out IPEndPoint mappedAddress)
{
    ushort ntoh(ushort Value)
    {
        return ((ushort)((((Value) & 0xff) << 8) | (((Value) & 0xff00) >> 8)));
    }

    mappedAddress = new IPEndPoint(IPAddress.None, 0);

    IPEndPoint stun_server = new IPEndPoint(Dns.GetHostEntry("stun.syncthing.net", AddressFamily.InterNetwork).AddressList.First(), 3478);

    Guid requestGUID = Guid.NewGuid();
    {
        MemoryStream ms = new MemoryStream();
        ms.Write(BitConverter.GetBytes((ushort)ntoh(1))); //MessageType
        ms.Write(BitConverter.GetBytes((ushort)ntoh(0))); //MessageLength
        ms.Write(requestGUID.ToByteArray());
        byte[] req = ms.ToArray();
        uc.Send(req, stun_server);
        ms.Close();
    }

    IPEndPoint changedAddress = null;
    Task<UdpReceiveResult> responseAsync = uc.ReceiveAsync();
    if (!responseAsync.Wait(1000))
    {
        return false;
    }
    if (!(responseAsync.Result.RemoteEndPoint.IPEndPointEquals(stun_server))) throw new InvalidOperationException("Packet received from a bad remote end point");
    byte[] buffer = responseAsync.Result.Buffer;
    {
        if (buffer.Length <= (sizeof(ushort) + sizeof(ushort) + sizeof(Guid))) throw new InvalidDataException();
        MemoryStream ms = new MemoryStream(buffer);
        ms.Seek(sizeof(ushort) + sizeof(ushort), SeekOrigin.Begin);
        Guid responseGUID = new Guid(
                BitConverter.ToUInt32([(byte)ms.ReadByte(), (byte)ms.ReadByte(), (byte)ms.ReadByte(), (byte)ms.ReadByte()]),
                BitConverter.ToUInt16([(byte)ms.ReadByte(), (byte)ms.ReadByte()]),
                BitConverter.ToUInt16([(byte)ms.ReadByte(), (byte)ms.ReadByte()]),
                (byte)ms.ReadByte(), (byte)ms.ReadByte(), (byte)ms.ReadByte(), (byte)ms.ReadByte(), (byte)ms.ReadByte(), (byte)ms.ReadByte(), (byte)ms.ReadByte(), (byte)ms.ReadByte()
            );
        if (requestGUID != responseGUID) throw new InvalidDataException("Wrong packet");
        for (; ; )
        {
            ushort attr = ntoh(BitConverter.ToUInt16([
                    (byte)ms.ReadByte(),
                    (byte)ms.ReadByte(),
                ]));
            if (attr == 0xffff) break;
            ushort length = ntoh(BitConverter.ToUInt16([
                    (byte)ms.ReadByte(),
                    (byte)ms.ReadByte(),
                ]));
            if (attr == 1)
            {
                ushort prot_fam = ntoh(BitConverter.ToUInt16([
                        (byte)ms.ReadByte(),
                    (byte)ms.ReadByte(),
                ]));
                if (prot_fam == 1)
                {
                    ushort port = ntoh(BitConverter.ToUInt16([(byte)ms.ReadByte(), (byte)ms.ReadByte()]));
                    IPAddress ip = new IPAddress([
                            (byte)ms.ReadByte(),
                       (byte)ms.ReadByte(),
                       (byte)ms.ReadByte(),
                       (byte)ms.ReadByte()
                        ]);
                    mappedAddress = new IPEndPoint(ip, port);
                }
            }
            else if (attr == 5)
            {
                ushort prot_fam = ntoh(BitConverter.ToUInt16([
                        (byte)ms.ReadByte(),
                    (byte)ms.ReadByte(),
                ]));
                if (prot_fam == 1)
                {
                    ushort port = ntoh(BitConverter.ToUInt16([(byte)ms.ReadByte(), (byte)ms.ReadByte()]));
                    IPAddress ip = new IPAddress([
                            (byte)ms.ReadByte(),
                       (byte)ms.ReadByte(),
                       (byte)ms.ReadByte(),
                       (byte)ms.ReadByte()
                        ]);
                    changedAddress = new IPEndPoint(ip, port);
                }
            }
            else
            {
                ms.Seek(length, SeekOrigin.Current);
            }
        }
    }
    if (changedAddress == null)
    {
        throw new InvalidOperationException("Bad stun server or stun server does not support change address");
    }

    Guid changeRequestGUID = Guid.NewGuid();
    {
        MemoryStream ms = new MemoryStream();
        ms.Write(BitConverter.GetBytes((ushort)ntoh(1))); //MessageType
        ms.Write(BitConverter.GetBytes((ushort)ntoh(8))); //MessageLength
        ms.Write(changeRequestGUID.ToByteArray());
        ms.Write(new byte[]
        {
            0x00,0x03,
            0x00,0x04,
            0x00,0x00,0x00,0b00000110
            //ChangeIP   0b00000100
            //ChangePort 0b00000010
        });
        byte[] req = ms.ToArray();
        uc.Send(req, stun_server);
        ms.Close();
    }

    Task<UdpReceiveResult> changeResponseAsync = uc.ReceiveAsync();
    if (!changeResponseAsync.Wait(1000))
    {
        return false;
    }
    if (!(changeResponseAsync.Result.RemoteEndPoint.IPEndPointEquals(changedAddress))) throw new InvalidOperationException("Packet received from a bad remote end point");
    byte[] resp = changeResponseAsync.Result.Buffer;
    {
        if (resp.Length <= (sizeof(ushort) + sizeof(ushort) + sizeof(Guid))) throw new InvalidDataException();
        MemoryStream ms = new MemoryStream(resp);
        ms.Seek(sizeof(ushort) + sizeof(ushort), SeekOrigin.Begin);
        Guid changeResponseGUID = new Guid(
                BitConverter.ToUInt32([(byte)ms.ReadByte(), (byte)ms.ReadByte(), (byte)ms.ReadByte(), (byte)ms.ReadByte()]),
                BitConverter.ToUInt16([(byte)ms.ReadByte(), (byte)ms.ReadByte()]),
                BitConverter.ToUInt16([(byte)ms.ReadByte(), (byte)ms.ReadByte()]),
                (byte)ms.ReadByte(), (byte)ms.ReadByte(), (byte)ms.ReadByte(), (byte)ms.ReadByte(), (byte)ms.ReadByte(), (byte)ms.ReadByte(), (byte)ms.ReadByte(), (byte)ms.ReadByte()
            );
        if (changeRequestGUID != changeResponseGUID) throw new InvalidDataException("Wrong packet");
    }

    return true;
}
```
2024年7月3日
