---
title: 设计一个数据缓冲队列（二）
date: 2020-01-20 22:14
categories: weaver
tags: C 数据结构
---

上一篇文章中我们给缓冲队列的设计打下了良好的基础。这次我们将在之前的基础上添加新的功能，并简单描述实现的思路。

在TCP协议中，拥塞控制的最重要和最基本的途径是窗口机制。比如说，在服务器向客户端发送大量数据的场景下，客户端会在每一个ACK包中包含两个信息：ACK number代表「到哪里为止的所有数据我都已经确定无误地收到了」，窗口大小表示「我有能力提供的缓冲区大小」。显然，作为一个力图保证效率的服务器端实现，在发送数据时，无论何时，数据段的左端都不应该小于ACK number——重复发送客户端已经宣称收到的数据是无意义的，数据段的右端不应该超出ACK number加上窗口大小，因为任何位置比ACK number更右的数据都（很）可能位于客户端的缓冲区内，而超出ACK number加上窗口大小的数据是无法在窗口大小的缓冲区内找到合适的存放位置的。

在weaver中，我们的目标不是实现一个协议栈单端（如TCP服务器端或客户端），而是实现一个协议栈双端，类似于监控程序。这也就意味着，当weaver收到了预期之外的数据时（比如乱序的数据段），它并不应该（也没有机会）像一个普通的协议单端那样，遵循协议的规范将异常事件向上层应用隐瞒（也就是发送重复的ACK number并「装作」没有接收到任何数据），而是要跳出协议，向上层应用报告异常，并自主地决定自身的行为。这颇有一点像是打破了第四面墙的故事人物。

那么，对于协议中存在的窗口，我们应该如何处理呢？按照上面的说法，我们的缓冲队列似乎应该无视窗口，仅仅是根据它的值报告有可能存在的异常事件。

但是事实上，我选择按照窗口指定的值对队列内已经缓冲的数据进行裁切。这样做的理由是，在理想情况下，我们维护的缓冲队列，其内容应该与客户端的内容保持一致——在客户端处理完weaver刚刚处理完的包以后。另一方面，weaver所记录的窗口数据来自于最后一个从客户端发送的ACK包，在我们处理下一个服务器端发送的数据包时开始，到客户端也处理同一个数据包为止，客户端的窗口范围*很有可能*保持不变（特殊情况接下来分析）。因此我们可以得到结论：如果一个数据包被我们判断为「超出窗口」，那么接下来收到它的客户端也会做出一样的判断而将其丢弃。因此，只要接下来客户端仍然能够将数据段凑齐，就意味着客户端一定通过协议的某种机制再一次（在窗口内）得到了之前被丢弃的数据段，换言之，我们也一样可以再次得到它。而另一方面，我们可以使用和客户端相同大小的缓冲区来维持我们的缓冲队列。

我们一定会与客户端采用完全一致的窗口范围处理数据段吗？由于我们只有在收到了客户端的ACK包以后才能得知客户端的最新状态，这种程度的滞后是无法消除的，因此，比如说在如下的情况下：我们处理完服务器端发来的1号数据包，然后客户端处理1号数据包时我们已经在处理服务器发来的2号数据包，那么客户端根据1号数据包所进行的自身状态更新我们暂时是看不到的，举一个极端的例子，如果客户端在处理1号包的过程中突然自闭，在对1号包的ACK中将自己的窗口大小设置为0，那么客户端将会丢弃接下来的2号包，而暂时不知道这一点的weaver则仍然按照原来的窗口大小将2号包缓存起来。

在这种情况下，这种不一致基本上是无害的：在根据1号包的ACK包更新了窗口大小后，在下次收到服务器发来的3号包时，我们就可以借此机会将2号包丢弃。因此我们仅仅是把一个事件的触发推迟了一个包。但是如果是一种反过来的情况，比如客户端在收到2号包（并将其丢弃）后又突然想开了，重新恢复了正常的窗口大小，那么不知道这一点的我们在处理3号包时，依然在采取之前的自闭零窗口，就会错误地将3号包丢弃。这是很严重的错误，因为客户端并没有将3号包丢弃，所以客户端不会以任何途径请求服务器重发3号包，weaver将再也没有机会将3号包重新取回。

严重的错误并不一定导致灾难性的后果。我们会丢失3号包，同时会陷入对3号包的等待中，但是很快，收到了3号包的客户端会在ACK包中将ACK number设置为3号包右端（或者更右），这次对窗口左端的记录将在接下来对服务器包的处理中起到作用——我们会意识到，虽然我们还在等3号包，但是它已经比左窗口还要左了——客户端已经不在等待它了。这会触发一个「数据段未收到就已经超出窗口」事件。接下来，队列不再等待3号包，weaver继续恢复正常运作。

最终，一次客户端的窗口增大在非常罕见地情况下，有可能触发一对假阳性事件：一个「收到的数据段超出右窗口」和一个「未收到的数据段超出左窗口」。通过这些事件，上层应用仍然有机会认清事实的真相并且收集到所有的数据段，并且最重要的是，后续的运行不会全盘皆错。

----

讲清楚为什么要为我们的队列添加窗口功能后，实际的实现相比之下就非常简单了。我们主要需要注意两点。第一是客户端的窗口可能非常大，比我们的缓存队列的缓冲区长度还要大，这时候就算一段数据没有超出客户端指定的右窗口，也可能会超出「我们的窗口」——队列的缓冲区长度。此时我们也不得不将这段数据丢弃，但是此时服务器端并没有做任何奇怪的事，完全是我们自己的问题，所以要通知上层应用「一切正常但是内存不够用了」。与之前类似，随着左窗口的逐渐向右移动，被丢弃的数据段空洞终将消失。当然了，如果是完全没有考虑数据的实现，则不存在这个问题。

还有一点是左窗口与队列偏移的相对关系。两者的含义非常相似但截然不同：左窗口代表着「应该收到这段数据的家伙觉得到这里为止的数据都已经凑齐了」，而队列偏移则代表「我把到此为止的数据都凑齐并交给上层应用了」。考虑到我们的角色设定，「应该收到数据的家伙」显然比「我」的优先级更高。因此，如果队列偏移比左窗口更靠右，则说明我们意外地比客户端收到了更多的数据——这预示着从我们到客户端的路上可能发生了丢包，此时收到的数据段如果比左窗口更靠左，那么这是一个「硬重传」：我们和客户端都同意这是重传；如果数据段比队列偏移更左，但是不比左窗口更左，那么这是一个「软重传」：这个包本身大概没有问题，反而是上一次传递这个数据段的包多半（在离开我们以后）出现了问题。如果左窗口比队列偏移靠右，那么情况比较不妙：两者之间的数据我们还在等待，但是客户端已经确认收到了。这意味着我们几乎不会再等到这段数据了，因此队列偏移一定要更新为左窗口值。比较省事的做法是报告上文中描述的「未收到的数据超出了左窗口」，更万全的做法则是，在每次发生「收到的数据段超出了右窗口」时，将被丢弃的数据段左右端记录下来，然后进行比较。如果发现从队列偏移到左窗口之间的数据的确被我们丢掉了，则向应用报告「不好意思我们信息有延迟/内存不够大所以不小心把这段数据给丢了」。

> 在前一种情况下，我们甚至还有机会完全隐瞒这一切——既不根据右窗口丢数据，又在此时装作这段数据刚收到，若无其事地交给上层应用——但是还是留给future work吧。
