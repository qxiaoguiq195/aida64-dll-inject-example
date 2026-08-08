使用吾爱破解 https://www.52pojie.cn/

991547436大佬修改过的版本

原贴

https://www.52pojie.cn/thread-2101242-1-1.html

大佬主页

https://www.52pojie.cn/?400305







原贴快照

[原创] 免patch激活软件教程——以aida64 business/extreme为例 [复制链接]
swigger

3

主题	
3

回帖	
23

积分
锋芒初露 Rank: 1
精华0威望0 点热心值19 点悬赏值0 点听众5贡献值0 点吾爱币72 CB违规0 次注册时间2026-3-13
收听TA
免patch激活软件教程
——以aida64 business/extreme等为例
Abstract
背景和基本思路
DLL劫持简介
选择要劫持的DLL
假 winmm.dll 代码详解
使用方法
电梯直达
跳转到指定楼层楼主
 发表于 2026-4-3 22:42 | 只看该作者 回帖奖励
本帖最后由 swigger 于 2026-4-3 22:53 编辑

免patch激活软件教程
——以aida64 business/extreme等为例
Abstract
本文介绍了一种破解软件的思路，利用该思路，可以不改目标软件任何文件一个字节实现激活，并且使得软件可以在一定时间内自由升级保持激活状态。

背景和基本思路
论坛上能看到很多破解教程，也看到有人为了减少修改，实现了RSA公钥只改一个字节的机制。然而，改一个字节也是改，改了就会影响数字签名，也可能导致杀软报错。有没有办法，一个字节也不改呢？当然，办法是有的。

早些天我在本论坛就放了一个某软件一字节也不改的激活方案，用过的朋友都说好。今天，我把这种方法拿出来，给大家讲解一下，对于初学者是很好的学习材料，对于高手，也算提供一种参考思路。

当然，一个字节也不改，又要修改目标软件的逻辑，方法就不多了。大致有几种：

利用驱动在内核级修改目标软件的逻辑
利用调试API强制介入修改目标软件
利用DLL劫持做目标软件的进程内Patch。
这几种方法我都试过，但在今天方法1可以直接淘汰了，因为缺点很明显，太难放驱动并且可能导致系统随时挂掉。方法2对于某些防dll劫持很严的软件来说是必要的。但市面上大多软件都没有做防dll劫持。dll劫持是一种很简单很方便的方案，适合大多数场景，本文就介绍这个。

注意：本文要求阅读者有一定的编程能力，对windows操作和破解有一些基本的了解，对C++有一定的理解能力，能使用Visual Studio编译出产品。本文用到的代码使用了C++20标准。阅读者需要能大致理解。当然，如果不理解，可以问AI。

DLL劫持简介
Windows 进程中exe要依赖大量的系统API和第三方API。当exe启动时，它会加载一些dll，而这些dll又会加载其它的dll。Windows维护了一个已知DLL列表，对于列表内的，它会直接去系统目录查找dll，而对于不在这个列表中的，它会首先尝试在exe所在目录加载。这样，当exe用到了xxx.dll时，如果我们在exe目录放一个假的xxx.dll，系统就会优先加载它，我们在假的xxx.dll中可以做一些事情，然后把API调用转到真的xxx.dll，就可以完成目标软件的修改了。

选择要劫持的DLL
我们以aida64为例，从地址 https://download2.aida64.com/aida64extreme825.zip 下载一个aida64 extreme（当然你把extreme改成business也行，对于本文接下来的内容不影响）。解开待用。打开 procmon64 ( https://learn.microsoft.com/en-us/sysinternals/downloads/procmon )，按图填入Filter（填时记得点Add)，点击OK。



双击启动 aida64.exe，看看结果里，有很多显示DLL找不到的条目。



可以看出，运行aida64时，它会尝试加载 mpr.dll,netapi32.dll,version.dll,winmm.dll等这些dll，由于这些dll不在已知DLL名单内，会首先尝试从exe所在目录加载。我以winmm.dll为例，我只需要在aida64目录放一个假的winmm.dll，它就会被加载，然后它调用真的winmm，这一切不就OK了？

假 winmm.dll 代码详解
为了制造一个假的winmm.dll，我们可以用vs2022，随便创建一个新的dll工程，编一个dll改名成winmm就行。

注意，我这里不再从0开始介绍一步步创建工程和加入代码，我把成品代码放在了：

https://github.com/swigger/aida64-dll-inject-example

我们直接用它来讲解。

我们知道dll加载后会调用DllMain函数，在dllmain.cpp中的该函数里，如果判断是进程启动加载，我首先调用了ui::start，这里是为了后续自动填入序列号。然后，我们判定主exe的状态，是不是upx版本，如果是，则hook 导入表中的 GetProcAddress，等upx在内存中解压完成后再调用我们的hook逻辑。否则的话用 QueueUserApc 启动一个异步回调，在dll全加载完，exe开始运行前，调用我们的hookFunc。

这个逻辑是因为默认的aida64.exe是upx压缩过的，我们要求一字节不改，当然要支持主exe是upx压缩的情况。不过在调试期间，我们可能已经用upx -d解出原始的aida64了，所以也要支持不是upx的情况。

工程中有一个winmm.def，导出了winmm的常见函数。这些是假winmm能工作的关键，我们的假winmm必须能支持真正的函数，支持方法就是跳到真函数去。我用脚本生成了winmm.def.asm，对于每一个假导出，它会跳转到一个函数，默认是跳到加载真导出地址再跳转到真地址的函数，在第一次找到真地址后，以后就只跳到真地址不再请求加载地址。

命名空间ui中的函数用于在弹出请求输入验证码那个窗口时，在内存中生成一个序列号，填上去。它首先一次性启动一个当前线程的CBT勾子，观察窗口变化的事件，当一个窗口第一次被激活时，判定是不是请求输入验证码的对话框，如果是，则延时10ms后，找到输入序列号的文本框，在内存中生成一个序列号，填好。

注意：为了尊重本论坛不放成品破解软件的规定，我这个工程里生成的序列号是假的，通不过验证，大家可以找到真实的算法，替换一下，就可以看到想要的结果。

在 hookFunc中，我们只做一件事，找到 LStrFromPChar 函数，然后 hook它。

为什么要这么做？

因为 aida64 有网络验证，如果你输入假的key，当场是通过了，但时不时的它就跳出一个窗口，说你的key是假的，你说气不气人。经过调试验证，它是在网络返回的xml中，识别特定信息然后弹框的。这个信息就是服务器在xml中放置的 <nod> <nod1>等信息。而服务器的xml会首先经过LStrFromPChar函数转成delphi可识别的字符串再进行后续操作的，所以只需要hook这个函数，把nodXX这些节点去掉，不就解决网验了？从此不就可以放心点在线升级？

要hook LStrFromPChar，有两个事要做：第一是找到它，第二是hook它到我们的自有函数。

要找到它并不难，因为aida64是用delphi做的，而这个LStrFromPChar又是delphi的内置函数，而delphi又不会有二进制级的变态优化，这就意味着LStrFromPChar函数的二进制码，在忽略重定位的情况下，是不会变的！因此，我们直接hardcode其前16字节，暴力在code段中查找一下，不就找到了？

然后是hook它。有一些现成的第三方hook可以用。比如微软的detours，比较知名的还有mini hook。我这里用了一个我自己手撸的direct hook，它采用破坏rax的方法来实现hook，适合在函数开始处进行hook。需要修改5或12字节。它虽然还有一些潜在的bug，也没有经过充分的测试，但胜在代码非常简短，并且很容易读懂和修改。暂时未实现unhook方法（很容易实现）。我在原代码中打包了，一并献给大家。

direct_hook(funcs[0], hook_LStrFromPChar, (void**)&trampo_LStrFromPChar);

这一句下来，我们把找到的 LStrFromPChar函数，也就是funcs[0]，强制让它跳转到我们的实现函数 hook_LStrFromPChar，在这个函数中，把nodXX给去掉，然后调用原始函数。

这就解决了网验的问题。

使用方法
编译出 dll后，把它拷贝到 aida64 所在目录，然后改名成winmm.dll，然后打开aida64，点击输入验证码，可以看到验证码自己填好啦。



注意：直接用本工程代码填的是假的，请大家自行修改keygen.cpp修正。

然后就注册成功了，再点击在线升级看看，静悄悄的，没有任何报假的弹窗！





免费评分
参与人数 4	吾爱币 +10	热心值 +4	收起理由
 Hmily	+ 7	+ 1	欢迎分析讨论交流，吾爱破解论坛有你更精彩！
 991547436	+ 2	+ 1	我很赞同！
 chengdragon	+ 1	+ 1	感谢分享
 whereismy		+ 1	我很赞同！
查看全部评分

收藏收藏29
 
免费评分免费评分
 
分享淘帖 
有用有用 
分享到朋友圈
发帖前要善用【论坛搜索】功能，那里可能会有你要找的答案或者已经有人发布过相同内容了，请勿重复发帖。

回复举报
991547436

27

主题	
307

回帖	
406

积分
出类拔萃 Rank: 4
热心值287 点悬赏值19 点吾爱币1322 CB注册时间2015-4-15
五年荣誉奖章十年荣誉奖章

收听TA
推荐
 发表于 2026-4-7 11:49 | 只看该作者
dllmain.cpp 的 strnicmp 会编译失败，需要改成 _strnicmp：

if (strnicmp(p, pt, len) == 0) return p;
-->
if (_strnicmp(p, pt, len) == 0) return p;

项目属性目标文件名可以直接改成winmm 省得重命名

然后把keygen.cpp替换成下面的代码

 复制代码 隐藏代码
#include "pch.h"
#include "keygen.h"

#pragma comment(lib, "version.lib")

namespace {
    constexpr char ALPHABET[] = "DY14UF3RHWCXLQB6IKJT9N5AGS2PM8VZ7E";

    bool base34_encode(uint32_t value, size_t width, std::string& out) {
        uint64_t limit = 1;
        for (size_t i = 0; i < width; ++i) {
            limit *= 34;
        }
        if (static_cast<uint64_t>(value) >= limit) {
            return false;
        }

        std::string chunk(width, 'D');
        for (size_t i = width; i-- > 0;) {
            chunk[i] = ALPHABET[value % 34];
            value /= 34;
        }
        out += chunk;
        return true;
    }

    uint32_t crc16_like(const std::string& text) {
        uint32_t value = 0;
        for (unsigned char c : text) {
            value ^= static_cast<uint32_t>(c) << 8;
            for (int i = 0; i < 8; ++i) {
                if (value & 0x8000u) {
                    value = ((value << 1) ^ 0x8201u) & 0xFFFFu;
                } else {
                    value = (value << 1) & 0xFFFFu;
                }
            }
        }
        return value;
    }

    char checksum_char(const std::string& first24) {
        uint32_t crc = crc16_like(first24) % 0x9987u;
        // base34_encode(crc, 3)[1] — middle char of 3-char encoding
        std::string tmp;
        base34_encode(crc, 3, tmp);
        return tmp[1];
    }

    uint32_t pack_base_date(int year, int month, int day) {
        return (static_cast<uint32_t>((year - 2003) & 0x1F) << 9)
             | (static_cast<uint32_t>(month & 0xF) << 5)
             | static_cast<uint32_t>(day & 0x1F);
    }

    uint32_t serial_limit_for_product(uint32_t product_id) {
        return (product_id == 1 || product_id == 4) ? 65533u : 776u;
    }

    uint32_t derive_serial(uint32_t product_id, uint32_t variant_code, uint32_t packed_date,
                           uint32_t expire_offset, uint32_t version_offset) {
        uint64_t mix = static_cast<uint64_t>(product_id)  * 0x45D
                     + static_cast<uint64_t>(variant_code) * 0x11
                     + static_cast<uint64_t>(packed_date)  * 3
                     + static_cast<uint64_t>(expire_offset)  * 5
                     + static_cast<uint64_t>(version_offset) * 7;
        return static_cast<uint32_t>(mix % serial_limit_for_product(product_id)) + 1;
    }

    uint32_t derive_seed(uint32_t product_id, uint32_t variant_code, uint32_t packed_date,
                         uint32_t expire_offset, uint32_t version_offset, uint32_t serial) {
        uint64_t mix = static_cast<uint64_t>(serial)
                     + static_cast<uint64_t>(product_id)  * 0x31
                     + static_cast<uint64_t>(variant_code) * 0x17
                     + static_cast<uint64_t>(packed_date)
                     + static_cast<uint64_t>(expire_offset)  * 7
                     + static_cast<uint64_t>(version_offset) * 11;
        return static_cast<uint32_t>(mix % (34u * 34u));
    }

    std::string group_key(const std::string& raw_key) {
        std::string grouped;
        grouped.reserve(raw_key.size() + raw_key.size() / 5);
        for (size_t i = 0; i < raw_key.size(); ++i) {
            if (i != 0 && i % 5 == 0) {
                grouped.push_back('-');
            }
            grouped.push_back(raw_key[i]);
        }
        return grouped;
    }

    bool contains_icase(const std::wstring& text, const wchar_t* needle) {
        return text.find(needle) != std::wstring::npos;
    }

    void to_lower_inplace(std::wstring& text) {
        std::transform(text.begin(), text.end(), text.begin(), [](wchar_t ch) {
            return static_cast<wchar_t>(towlower(ch));
        });
    }

    std::wstring query_version_string(const void* version_info, WORD language, WORD codepage, const wchar_t* name) {
        wchar_t sub_block[64]{};
        swprintf_s(sub_block, L"\\StringFileInfo\\%04x%04x\\%s", language, codepage, name);

        void* value = nullptr;
        UINT value_size = 0;
        if (!VerQueryValueW(version_info, sub_block, &value, &value_size) || value == nullptr || value_size == 0) {
            return {};
        }
        return std::wstring(static_cast<const wchar_t*>(value));
    }

    std::wstring read_version_info_product_name(HMODULE mod) {
        wchar_t path[MAX_PATH]{};
        const DWORD path_len = GetModuleFileNameW(mod, path, static_cast<DWORD>(std::size(path)));
        if (path_len == 0 || path_len >= std::size(path)) {
            return {};
        }

        DWORD handle = 0;
        const DWORD version_size = GetFileVersionInfoSizeW(path, &handle);
        if (version_size == 0) {
            return {};
        }

        std::vector<BYTE> version_info(version_size);
        if (!GetFileVersionInfoW(path, 0, version_size, version_info.data())) {
            return {};
        }

        struct Translation {
            WORD language;
            WORD codepage;
        };

        void* translation_ptr = nullptr;
        UINT translation_size = 0;
        if (VerQueryValueW(version_info.data(), L"\\VarFileInfo\\Translation", &translation_ptr, &translation_size)
            && translation_ptr != nullptr
            && translation_size >= sizeof(Translation)) {
            const auto* translations = static_cast<const Translation*>(translation_ptr);
            const size_t translation_count = translation_size / sizeof(Translation);
            for (size_t i = 0; i < translation_count; ++i) {
                std::wstring product_name = query_version_string(
                    version_info.data(), translations[i].language, translations[i].codepage, L"ProductName");
                if (!product_name.empty()) {
                    return product_name;
                }
            }
        }

        for (const Translation fallback : { Translation{ 0x0409, 0x04B0 }, Translation{ 0x0409, 1200 } }) {
            std::wstring product_name = query_version_string(version_info.data(), fallback.language, fallback.codepage, L"ProductName");
            if (!product_name.empty()) {
                return product_name;
            }
        }

        return {};
    }
}

ProductType detect_product_type()
{
    const HMODULE mod = GetModuleHandleW(nullptr);
    std::wstring product_name = read_version_info_product_name(mod);
    to_lower_inplace(product_name);

    if (contains_icase(product_name, L"business")) {
        return VER_BUSINESS;
    }
    if (contains_icase(product_name, L"extreme")) {
        return VER_EXTREME;
    }
    if (contains_icase(product_name, L"engineer")) {
        return VER_ENGINEER;
    }
    if (contains_icase(product_name, L"network")) {
        return VER_NETWORK;
    }
    // may add some more checks here. just fallback.
    return VER_BUSINESS;
}

std::string gen_key(ProductType pt, uint32_t subtype, std::time_t issue_date, std::time_t expire, std::time_t support_due)
{
    const uint32_t product_id = static_cast<uint32_t>(pt);
    if (product_id == 0 || product_id > 999) {
        return {};
    }

    // For product_id == 1 (Business), variant_code is always fixed to 100.
    const uint32_t variant_code = (product_id == 1) ? 100u : subtype;

    // Resolve base date (issue_date==0 means today).
    const std::time_t base_ts = issue_date ? issue_date : std::time(nullptr);
    struct tm tm_base{};
    localtime_s(&tm_base, &base_ts);
    const int b_year  = tm_base.tm_year + 1900;
    const int b_month = tm_base.tm_mon  + 1;
    const int b_day   = tm_base.tm_mday;

    // expire_offset and version_offset in days from base_date.
    constexpr uint32_t DEFAULT_OFFSET = 3652u;
    const uint32_t expire_offset = (expire > base_ts)
        ? std::min<uint32_t>(static_cast<uint32_t>((expire - base_ts) / 86400), DEFAULT_OFFSET)
        : DEFAULT_OFFSET;
    const uint32_t version_offset = (support_due > base_ts)
        ? std::min<uint32_t>(std::max<uint32_t>(static_cast<uint32_t>((support_due - base_ts) / 86400), 1u), DEFAULT_OFFSET)
        : DEFAULT_OFFSET;

    constexpr uint32_t FIELD_5_6 = 6u;
    constexpr uint32_t FIELD_7_8 = 1u;

    const uint32_t packed_date    = pack_base_date(b_year, b_month, b_day);
    const uint32_t serial         = derive_serial(product_id, variant_code, packed_date, expire_offset, version_offset);
    const uint32_t seed           = derive_seed(product_id, variant_code, packed_date, expire_offset, version_offset, serial);
    const uint32_t seed_low       = seed & 0xFFu;

    std::string first24;
    first24.reserve(24);
    if (!base34_encode(product_id   ^ seed_low ^ 0xBFu,   2, first24)
     || !base34_encode(variant_code ^ seed_low ^ 0xEDu,   2, first24)
     || !base34_encode(FIELD_5_6   ^ seed_low ^ 0x77u,   2, first24)
     || !base34_encode(FIELD_7_8   ^ seed_low ^ 0xDFu,   2, first24)
     || !base34_encode(serial      ^ seed     ^ 0x4755u,  4, first24)
     || !base34_encode(packed_date ^ seed     ^ 0x7CC1u,  4, first24)
     || !base34_encode(expire_offset  ^ seed_low ^ 0x3FDu,  3, first24)
     || !base34_encode(version_offset ^ seed_low ^ 0x935u,  3, first24)
     || !base34_encode(seed,                               2, first24)) {
        return {};
    }

    std::string raw_key = first24;
    raw_key += checksum_char(first24);
    return group_key(raw_key);
}

免费评分
参与人数 1	吾爱币 +1	热心值 +1	收起理由
 qxiaoguiq	+ 1	+ 1	我很赞同！
查看全部评分

https://api.xhboke.com/news/
【吾爱破解论坛总版规】 - [让你充分了解吾爱破解论坛行为规则]
回复 支持 3免费评分 举报
fishs

7

主题	
65

回帖	
46

积分
锋芒初露 Rank: 1
热心值31 点悬赏值1 点吾爱币897 CB注册时间2025-3-13
沙发
 发表于 2026-4-5 22:53 | 只看该作者
吾爱破解论坛没有任何官方QQ群，禁止留联系方式，禁止任何商业交易。
牛人，是不是一些需要VIP会员登录的软件也可以用此方法跳过网检？
如何升级？如何获得积分？积分对应解释说明！
回复 支持免费评分 举报
wipjjj

2

主题	
254

回帖	
45

积分
锋芒初露 Rank: 1
热心值16 点吾爱币3470 CB注册时间2019-3-13
3#
 发表于 2026-4-6 09:28 | 只看该作者
《站点帮助文档》有什么问题来这里看看吧，这里有你想知道的内容！
牛！学习了。
呼吁大家发布原创作品添加吾爱破解论坛标识！
回复 支持免费评分 举报
douyacai

2

主题	
162

回帖	
16

积分
锋芒初露 Rank: 1
吾爱币422 CB注册时间2026-3-13
4#
 发表于 2026-4-6 11:07 | 只看该作者
写的太好了，还可以自己练练手，学习了。
如何快速判断一个文件是否为病毒！
回复 支持免费评分 举报
twshe

10

主题	
147

回帖	
18

积分
锋芒初露 Rank: 1
热心值1 点悬赏值1 点吾爱币3308 CB注册时间2012-2-13
5#
 发表于 2026-4-6 14:55 | 只看该作者
学习一下，这种方式比较好，保留原汁原味，可升级！
回复 支持免费评分 举报
b1062

1

主题	
34

回帖	
4

积分
锋芒初露 Rank: 1
吾爱币50 CB注册时间2026-3-13
6#
 发表于 2026-4-6 16:00 | 只看该作者
这个厉害了
回复 支持免费评分 举报
qq243241

0

主题	
51

回帖	
5

积分
锋芒初露 Rank: 1
吾爱币313 CB注册时间2013-11-11
7#
 发表于 2026-4-6 21:15 | 只看该作者
另辟蹊径的思路，感谢分享学习
回复 支持免费评分 举报
alanlyf

0

主题	
55

回帖	
6

积分
锋芒初露 Rank: 1
吾爱币339 CB注册时间2026-3-13
8#
 发表于 2026-4-7 02:19 | 只看该作者
思路很棒，很有启发意义，收下了
回复 支持免费评分 举报
ttttyyyy

1

主题	
246

回帖	
25

积分
锋芒初露 Rank: 1
吾爱币1536 CB注册时间2016-11-11
9#
 发表于 2026-4-7 07:52 | 只看该作者
感谢大佬分享！
回复 支持免费评分 举报
freecat

1

主题	
173

回帖	
19

积分
锋芒初露 Rank: 1
热心值1 点吾爱币4256 CB注册时间2008-4-11
10#
 发表于 2026-4-7 09:31 | 只看该作者
不错 学习了
回复 支持免费评分 举报
 楼主: swigger
上一主题 下一主题
收起左侧
[原创] 免patch激活软件教程——以aida64 business/extreme为例 [复制链接]
991547436

27

主题	
307

回帖	
406

积分
出类拔萃 Rank: 4
热心值287 点悬赏值19 点吾爱币1322 CB注册时间2015-4-15
五年荣誉奖章十年荣誉奖章

收听TA
11#
 发表于 2026-4-7 11:49 | 只看该作者
dllmain.cpp 的 strnicmp 会编译失败，需要改成 _strnicmp：

if (strnicmp(p, pt, len) == 0) return p;
-->
if (_strnicmp(p, pt, len) == 0) return p;

项目属性目标文件名可以直接改成winmm 省得重命名

然后把keygen.cpp替换成下面的代码

#include "pch.h"
#include "keygen.h"

#pragma comment(lib, "version.lib")

namespace {
    constexpr char ALPHABET[] = "DY14UF3RHWCXLQB6IKJT9N5AGS2PM8VZ7E";

    bool base34_encode(uint32_t value, size_t width, std::string& out) {
        uint64_t limit = 1;
        for (size_t i = 0; i < width; ++i) {
            limit *= 34;
        }
        if (static_cast<uint64_t>(value) >= limit) {
            return false;
        }

        std::string chunk(width, 'D');
        for (size_t i = width; i-- > 0;) {
            chunk[i] = ALPHABET[value % 34];
            value /= 34;
        }
        out += chunk;
        return true;
    }

    uint32_t crc16_like(const std::string& text) {
        uint32_t value = 0;
        for (unsigned char c : text) {
            value ^= static_cast<uint32_t>(c) << 8;
            for (int i = 0; i < 8; ++i) {
                if (value & 0x8000u) {
                    value = ((value << 1) ^ 0x8201u) & 0xFFFFu;
                } else {
                    value = (value << 1) & 0xFFFFu;
                }
            }
        }
        return value;
    }

    char checksum_char(const std::string& first24) {
        uint32_t crc = crc16_like(first24) % 0x9987u;
        // base34_encode(crc, 3)[1] — middle char of 3-char encoding
        std::string tmp;
        base34_encode(crc, 3, tmp);
        return tmp[1];
    }

    uint32_t pack_base_date(int year, int month, int day) {
        return (static_cast<uint32_t>((year - 2003) & 0x1F) << 9)
             | (static_cast<uint32_t>(month & 0xF) << 5)
             | static_cast<uint32_t>(day & 0x1F);
    }

    uint32_t serial_limit_for_product(uint32_t product_id) {
        return (product_id == 1 || product_id == 4) ? 65533u : 776u;
    }

    uint32_t derive_serial(uint32_t product_id, uint32_t variant_code, uint32_t packed_date,
                           uint32_t expire_offset, uint32_t version_offset) {
        uint64_t mix = static_cast<uint64_t>(product_id)  * 0x45D
                     + static_cast<uint64_t>(variant_code) * 0x11
                     + static_cast<uint64_t>(packed_date)  * 3
                     + static_cast<uint64_t>(expire_offset)  * 5
                     + static_cast<uint64_t>(version_offset) * 7;
        return static_cast<uint32_t>(mix % serial_limit_for_product(product_id)) + 1;
    }

    uint32_t derive_seed(uint32_t product_id, uint32_t variant_code, uint32_t packed_date,
                         uint32_t expire_offset, uint32_t version_offset, uint32_t serial) {
        uint64_t mix = static_cast<uint64_t>(serial)
                     + static_cast<uint64_t>(product_id)  * 0x31
                     + static_cast<uint64_t>(variant_code) * 0x17
                     + static_cast<uint64_t>(packed_date)
                     + static_cast<uint64_t>(expire_offset)  * 7
                     + static_cast<uint64_t>(version_offset) * 11;
        return static_cast<uint32_t>(mix % (34u * 34u));
    }

    std::string group_key(const std::string& raw_key) {
        std::string grouped;
        grouped.reserve(raw_key.size() + raw_key.size() / 5);
        for (size_t i = 0; i < raw_key.size(); ++i) {
            if (i != 0 && i % 5 == 0) {
                grouped.push_back('-');
            }
            grouped.push_back(raw_key[i]);
        }
        return grouped;
    }

    bool contains_icase(const std::wstring& text, const wchar_t* needle) {
        return text.find(needle) != std::wstring::npos;
    }

    void to_lower_inplace(std::wstring& text) {
        std::transform(text.begin(), text.end(), text.begin(), [](wchar_t ch) {
            return static_cast<wchar_t>(towlower(ch));
        });
    }

    std::wstring query_version_string(const void* version_info, WORD language, WORD codepage, const wchar_t* name) {
        wchar_t sub_block[64]{};
        swprintf_s(sub_block, L"\\StringFileInfo\\%04x%04x\\%s", language, codepage, name);

        void* value = nullptr;
        UINT value_size = 0;
        if (!VerQueryValueW(version_info, sub_block, &value, &value_size) || value == nullptr || value_size == 0) {
            return {};
        }
        return std::wstring(static_cast<const wchar_t*>(value));
    }

    std::wstring read_version_info_product_name(HMODULE mod) {
        wchar_t path[MAX_PATH]{};
        const DWORD path_len = GetModuleFileNameW(mod, path, static_cast<DWORD>(std::size(path)));
        if (path_len == 0 || path_len >= std::size(path)) {
            return {};
        }

        DWORD handle = 0;
        const DWORD version_size = GetFileVersionInfoSizeW(path, &handle);
        if (version_size == 0) {
            return {};
        }

        std::vector<BYTE> version_info(version_size);
        if (!GetFileVersionInfoW(path, 0, version_size, version_info.data())) {
            return {};
        }

        struct Translation {
            WORD language;
            WORD codepage;
        };

        void* translation_ptr = nullptr;
        UINT translation_size = 0;
        if (VerQueryValueW(version_info.data(), L"\\VarFileInfo\\Translation", &translation_ptr, &translation_size)
            && translation_ptr != nullptr
            && translation_size >= sizeof(Translation)) {
            const auto* translations = static_cast<const Translation*>(translation_ptr);
            const size_t translation_count = translation_size / sizeof(Translation);
            for (size_t i = 0; i < translation_count; ++i) {
                std::wstring product_name = query_version_string(
                    version_info.data(), translations[i].language, translations[i].codepage, L"ProductName");
                if (!product_name.empty()) {
                    return product_name;
                }
            }
        }

        for (const Translation fallback : { Translation{ 0x0409, 0x04B0 }, Translation{ 0x0409, 1200 } }) {
            std::wstring product_name = query_version_string(version_info.data(), fallback.language, fallback.codepage, L"ProductName");
            if (!product_name.empty()) {
                return product_name;
            }
        }

        return {};
    }
}

ProductType detect_product_type()
{
    const HMODULE mod = GetModuleHandleW(nullptr);
    std::wstring product_name = read_version_info_product_name(mod);
    to_lower_inplace(product_name);

    if (contains_icase(product_name, L"business")) {
        return VER_BUSINESS;
    }
    if (contains_icase(product_name, L"extreme")) {
        return VER_EXTREME;
    }
    if (contains_icase(product_name, L"engineer")) {
        return VER_ENGINEER;
    }
    if (contains_icase(product_name, L"network")) {
        return VER_NETWORK;
    }
    // may add some more checks here. just fallback.
    return VER_BUSINESS;
}

std::string gen_key(ProductType pt, uint32_t subtype, std::time_t issue_date, std::time_t expire, std::time_t support_due)
{
    const uint32_t product_id = static_cast<uint32_t>(pt);
    if (product_id == 0 || product_id > 999) {
        return {};
    }

    // For product_id == 1 (Business), variant_code is always fixed to 100.
    const uint32_t variant_code = (product_id == 1) ? 100u : subtype;

    // Resolve base date (issue_date==0 means today).
    const std::time_t base_ts = issue_date ? issue_date : std::time(nullptr);
    struct tm tm_base{};
    localtime_s(&tm_base, &base_ts);
    const int b_year  = tm_base.tm_year + 1900;
    const int b_month = tm_base.tm_mon  + 1;
    const int b_day   = tm_base.tm_mday;

    // expire_offset and version_offset in days from base_date.
    constexpr uint32_t DEFAULT_OFFSET = 3652u;
    const uint32_t expire_offset = (expire > base_ts)
        ? std::min<uint32_t>(static_cast<uint32_t>((expire - base_ts) / 86400), DEFAULT_OFFSET)
        : DEFAULT_OFFSET;
    const uint32_t version_offset = (support_due > base_ts)
        ? std::min<uint32_t>(std::max<uint32_t>(static_cast<uint32_t>((support_due - base_ts) / 86400), 1u), DEFAULT_OFFSET)
        : DEFAULT_OFFSET;

    constexpr uint32_t FIELD_5_6 = 6u;
    constexpr uint32_t FIELD_7_8 = 1u;

    const uint32_t packed_date    = pack_base_date(b_year, b_month, b_day);
    const uint32_t serial         = derive_serial(product_id, variant_code, packed_date, expire_offset, version_offset);
    const uint32_t seed           = derive_seed(product_id, variant_code, packed_date, expire_offset, version_offset, serial);
    const uint32_t seed_low       = seed & 0xFFu;

    std::string first24;
    first24.reserve(24);
    if (!base34_encode(product_id   ^ seed_low ^ 0xBFu,   2, first24)
     || !base34_encode(variant_code ^ seed_low ^ 0xEDu,   2, first24)
     || !base34_encode(FIELD_5_6   ^ seed_low ^ 0x77u,   2, first24)
     || !base34_encode(FIELD_7_8   ^ seed_low ^ 0xDFu,   2, first24)
     || !base34_encode(serial      ^ seed     ^ 0x4755u,  4, first24)
     || !base34_encode(packed_date ^ seed     ^ 0x7CC1u,  4, first24)
     || !base34_encode(expire_offset  ^ seed_low ^ 0x3FDu,  3, first24)
     || !base34_encode(version_offset ^ seed_low ^ 0x935u,  3, first24)
     || !base34_encode(seed,                               2, first24)) {
        return {};
    }

    std::string raw_key = first24;
    raw_key += checksum_char(first24);
    return group_key(raw_key);
}

免费评分
参与人数 1	吾爱币 +1	热心值 +1	收起理由
 qxiaoguiq	+ 1	+ 1	我很赞同！
查看全部评分

https://api.xhboke.com/news/
发帖前要善用【论坛搜索】功能，那里可能会有你要找的答案或者已经有人发布过相同内容了，请勿重复发帖。

回复 支持 3免费评分 举报
jsqdcdh

0

主题	
34

回帖	
3

积分
锋芒初露 Rank: 1
吾爱币958 CB注册时间2024-7-21
12#
 发表于 2026-4-7 13:01 | 只看该作者
感谢分享，有用
【吾爱破解论坛总版规】 - [让你充分了解吾爱破解论坛行为规则]
回复 支持免费评分 举报
Farkeasssauf

0

主题	
14

回帖	
1

积分
锋芒初露 Rank: 1
吾爱币265 CB注册时间2026-3-13
13#
 发表于 2026-4-11 15:24 | 只看该作者
吾爱破解论坛没有任何官方QQ群，禁止留联系方式，禁止任何商业交易。
学习一下  感谢分享
如何升级？如何获得积分？积分对应解释说明！
回复 支持免费评分 举报
mxn007

0

主题	
61

回帖	
6

积分
锋芒初露 Rank: 1
吾爱币1991 CB注册时间2023-11-11
14#
 发表于 2026-4-14 05:45 | 只看该作者
《站点帮助文档》有什么问题来这里看看吧，这里有你想知道的内容！

学习一下
呼吁大家发布原创作品添加吾爱破解论坛标识！
回复 支持免费评分 举报
xygzxygz

5

主题	
30

回帖	
14

积分
锋芒初露 Rank: 1
热心值9 点吾爱币38 CB注册时间2026-3-13
15#
 发表于 2026-4-14 08:53 | 只看该作者
方法很好，要求有点高，得加油学习了
如何快速判断一个文件是否为病毒！
回复 支持免费评分 举报
smartfrog

1

主题	
38

回帖	
4

积分
锋芒初露 Rank: 1
吾爱币442 CB注册时间2020-9-26
16#
 发表于 2026-4-14 14:19 | 只看该作者
思路不错，有借鉴意义
回复 支持免费评分 举报
Hmily论坛大牛

3089

主题	
4万

回帖	
7万

积分
人在江湖 Rank: 16Rank: 16Rank: 16Rank: 16
精华9威望190 点热心值34965 点悬赏值146 点贡献值15638 点吾爱币7728 CB注册时间2008-3-13
十年荣誉奖章宣传大使奖缉毒先锋勋章十五年荣誉奖章

收听TA
17#
 发表于 2026-4-16 11:50 | 只看该作者
这种方式有个老名字叫内存补丁，api hook修改也是范围内吧。
发帖求助前要善用【论坛搜索】功能，那里可能会有你要找的答案。
如果你在论坛求助问题，并且已经从坛友或者管理的回复中解决了问题，请把帖子标题加上【已解决】！
如何回报帮助你解决问题的坛友，一个好办法就是点击帖子下方评分按钮给对方加【热心值】，加分不会扣除自己的积分，做一个热心并受欢迎的人
回复 支持免费评分 举报
gra1108

0

主题	
15

回帖	
2

积分
锋芒初露 Rank: 1
吾爱币96 CB注册时间2026-3-13
18#
 发表于 2026-4-17 16:08 | 只看该作者
收藏学习，感谢实力大神的分享
回复 支持免费评分 举报
gyx1236230

0

主题	
43

回帖	
4

积分
锋芒初露 Rank: 1
吾爱币1322 CB注册时间2021-7-21
19#
 发表于 2026-4-17 16:13 | 只看该作者

厉害,学习学习
回复 支持免费评分 举报
se2537

0

主题	
22

回帖	
2

积分
锋芒初露 Rank: 1
吾爱币27 CB注册时间2025-11-11
20#
 发表于 2026-4-17 16:45 | 只看该作者
牛！学习了。
回复 支持免费评分 举报
 楼主: swigger
上一主题 下一主题
收起左侧
[原创] 免patch激活软件教程——以aida64 business/extreme为例 [复制链接]
gyx1236230

0

主题	
43

回帖	
4

积分
锋芒初露 Rank: 1
吾爱币1322 CB注册时间2021-7-21
21#
 发表于 2026-4-17 16:13 | 只看该作者

厉害,学习学习
发帖前要善用【论坛搜索】功能，那里可能会有你要找的答案或者已经有人发布过相同内容了，请勿重复发帖。

回复 支持免费评分 举报
se2537

0

主题	
22

回帖	
2

积分
锋芒初露 Rank: 1
吾爱币27 CB注册时间2025-11-11
22#
 发表于 2026-4-17 16:45 | 只看该作者
牛！学习了。
【吾爱破解论坛总版规】 - [让你充分了解吾爱破解论坛行为规则]
回复 支持免费评分 举报
Bootybay

0

主题	
35

回帖	
4

积分
锋芒初露 Rank: 1
吾爱币181 CB注册时间2018-7-21
23#
 发表于 2026-4-17 23:02 | 只看该作者
吾爱破解论坛没有任何官方QQ群，禁止留联系方式，禁止任何商业交易。
大佬牛逼，来学习一下
如何升级？如何获得积分？积分对应解释说明！
回复 支持免费评分 举报
aliuwr

0

主题	
28

回帖	
3

积分
锋芒初露 Rank: 1
吾爱币132 CB注册时间2011-7-11
24#
 发表于 2026-4-17 23:15 | 只看该作者
《站点帮助文档》有什么问题来这里看看吧，这里有你想知道的内容！
思路很合理
呼吁大家发布原创作品添加吾爱破解论坛标识！
回复 支持免费评分 举报
qxiaoguiq

0

主题	
39

回帖	
4

积分
锋芒初露 Rank: 1
吾爱币154 CB注册时间2024-3-13
25#
 发表于 2026-8-8 19:25 | 只看该作者 |自己
大佬大佬编译完成加到目录后出现应用无法正常启动怎么办
如何快速判断一个文件是否为病毒！
回复 编辑 支持免费评分
qxiaoguiq

0

主题	
39

回帖	
4

积分
锋芒初露 Rank: 1
吾爱币154 CB注册时间2024-3-13
26#
 发表于 2026-8-8 19:27 | 只看该作者 |自己
991547436 发表于 2026-4-7 11:49
[md]`dllmain.cpp` 的 `strnicmp` 会编译失败，需要改成 `_strnicmp`：

if (strnicmp(p, pt, len) == 0) ...


大佬大佬编译完成加到目录后出现应用无法正常启动怎么办
回复 编辑 支持免费评分
qxiaoguiq

0

主题	
39

回帖	
4

积分
锋芒初露 Rank: 1
吾爱币154 CB注册时间2024-3-13
27#
 发表于 2026-8-8 19:51 | 只看该作者 |自己
991547436 发表于 2026-4-7 11:49
[md]`dllmain.cpp` 的 `strnicmp` 会编译失败，需要改成 `_strnicmp`：

if (strnicmp(p, pt, len) == 0) ...

我知道问题了必须本地编译，不能使用github自动编译，谢谢大佬
回复 编辑 支持
