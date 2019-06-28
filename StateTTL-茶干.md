---


---

<h2 id="flink-1.8.0中的状态生存时间特性：如何自动清理应用程序的状态">Flink 1.8.0中的状态生存时间特性：如何自动清理应用程序的状态</h2>
<p>对于许多状态流式计算程序来说，一个常见的需求是自动清理应用程序的状态，以便有效地控制状态大小，或者控制程序访问状态的有效时间（例如受限于诸如GDPR等法律条规）。Apache Flink自1.6.0版本引入了状态的生存时间（time-to-live，TTL）功能，使得应用程序的状态清理和有效的状态大小管理成为可能。</p>
<p>在本文中，我们将讨论引入状态生存时间特性的动机并讨论其相关用例。此外，我们还将演示如何使用和配置该特性。同时，我们将会解释Flink如何借用状态生存时间特性在内部管理状态，并对Flink 1.8.0中该功能引入的相关新特性进行一些展示。本文章最后对未来的改进和扩展作了展望。</p>
<h3 id="状态的暂时性">状态的暂时性</h3>
<p>有两个主要原因可以解释为什么状态只应该维持有限的时间。让我们先设想一个Flink应用程序，它接收用户登录事件流，并为每个用户存储上一次登录时进行相关事件信息和时间戳，以改善频繁访问用户的体验。</p>
<ul>
<li><strong>控制状态的大小。</strong> 状态生存时间特性的主要使用场景，就是能够有效地管理不断增长的状态大小。通常情况下，数据只需要暂时保存，例如用户处在一次网络访问连接会话中。当用户访问事件结束时，我们实际上就没有必要保存该用户的状态，来减少无谓的状态存储空间占用。Flink 1.8.0引入的基于生存时间的旧状态后台清理机制，使得我们能够自动地对无用数据进行清理。此前，应用程序开发人员必须采取额外的操作并显式地删除无用状态以释放存储空间。这种手动清理过程不仅容易出错，而且效率低下。以我们上述用户登录的案例为例，我们就不再必须额外存储上次登录的时间戳，因为这些不活跃用户的相关信息会被自动过期清理掉。</li>
<li><strong>符合数据保护和敏感数据的要求。</strong> 随着数据隐私法规的发展（例如欧盟颁布的通用数据保护法规GDPR），遵守此类法规的相关要求，或将将数据进行敏感处理已经成为许多应用程序的首要任务。此类使用场景的的一个典型案例，就需要仅在特定时间段内保存数据并防止其后可以再次访问改数据。这对于为客户提供短期服务的公司来说是一个常见的挑战。状态生存时间这一特性，就保证了应用程序仅可以在有限时间内进行访问，有助于遵守数据保护法规。</li>
</ul>
<p>这两个需求都可以通过状态生存时间来解决，这个功能可以周期性地、但连续地删除一个键的状态，一旦它变得不必要或不重要，并且不再需要将它保存在存储中时。</p>
<h3 id="对应用状态的持续清理">对应用状态的持续清理</h3>
<p>Apache Flink的1.6.0版本引入了状态生存时间特性。它使流处理应用程序的开发人员能够配置算子的状态，使其在定义的超时（生存时间）后过期并被清除。在Flink 1.8.0中，该功能得到了进一步扩展，对RocksDB和内存堆状态后端（<code>FsStateBackend</code>和<code>MemoryStateBackend</code>）的旧数据进行连续清理，从而实现根据相关设置对旧数据条目进行连续清理的功能。<br>
在Flink的<code>DataStream</code> API中，应用程序状态是由状态描述符（state descriptor）来定义的。状态生存时间是通过将<code>StateTtlConfiguration</code>对象传递给状态描述符来配置的。下面的Java示例演示了如何创建状态生存时间的配置，并将其提供给状态描述符，该状态描述符将用户的上次登录时间保存为<code>Long</code>值：</p>
<pre class=" language-java"><code class="prism  language-java"><span class="token keyword">import</span> org<span class="token punctuation">.</span>apache<span class="token punctuation">.</span>flink<span class="token punctuation">.</span>api<span class="token punctuation">.</span>common<span class="token punctuation">.</span>state<span class="token punctuation">.</span>StateTtlConfig<span class="token punctuation">;</span>
<span class="token keyword">import</span> org<span class="token punctuation">.</span>apache<span class="token punctuation">.</span>flink<span class="token punctuation">.</span>api<span class="token punctuation">.</span>common<span class="token punctuation">.</span>time<span class="token punctuation">.</span>Time<span class="token punctuation">;</span>
<span class="token keyword">import</span> org<span class="token punctuation">.</span>apache<span class="token punctuation">.</span>flink<span class="token punctuation">.</span>api<span class="token punctuation">.</span>common<span class="token punctuation">.</span>state<span class="token punctuation">.</span>ValueStateDescriptor<span class="token punctuation">;</span>

StateTtlConfig ttlConfig <span class="token operator">=</span> StateTtlConfig
    <span class="token punctuation">.</span><span class="token function">newBuilder</span><span class="token punctuation">(</span>Time<span class="token punctuation">.</span><span class="token function">days</span><span class="token punctuation">(</span><span class="token number">7</span><span class="token punctuation">)</span><span class="token punctuation">)</span>
    <span class="token punctuation">.</span><span class="token function">setUpdateType</span><span class="token punctuation">(</span>StateTtlConfig<span class="token punctuation">.</span>UpdateType<span class="token punctuation">.</span>OnCreateAndWrite<span class="token punctuation">)</span>
    <span class="token punctuation">.</span><span class="token function">setStateVisibility</span><span class="token punctuation">(</span>StateTtlConfig<span class="token punctuation">.</span>StateVisibility<span class="token punctuation">.</span>NeverReturnExpired<span class="token punctuation">)</span>
    <span class="token punctuation">.</span><span class="token function">build</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>

ValueStateDescriptor<span class="token operator">&lt;</span>Long<span class="token operator">&gt;</span> lastUserLogin <span class="token operator">=</span>
    <span class="token keyword">new</span> <span class="token class-name">ValueStateDescriptor</span><span class="token operator">&lt;</span><span class="token operator">&gt;</span><span class="token punctuation">(</span><span class="token string">"lastUserLogin"</span><span class="token punctuation">,</span> Long<span class="token punctuation">.</span><span class="token keyword">class</span><span class="token punctuation">)</span><span class="token punctuation">;</span>

lastUserLogin<span class="token punctuation">.</span><span class="token function">enableTimeToLive</span><span class="token punctuation">(</span>ttlConfig<span class="token punctuation">)</span><span class="token punctuation">;</span>
</code></pre>
<p>Flink提供了多个选项来配置状态生存时间的行为:</p>
<ul>
<li><strong>什么时候重置生存时间？</strong> 默认情况下，当状态被修改时，生存时间就会被更新。我们也可以在读访问状态时更新相关项的生存时间，但这样要花费额外的写操作来更新时间戳。</li>
<li><strong>已经过期的数据是否可以访问？</strong> 状态生存时间机制使用的是惰性策略来清除过期状态。这可能导致应用程序会尝试读取过期但尚未删除的状态。用户可以配置对这样的读取请求是否返回过期状态。无论哪种情况，过期状态都会在之后立即被删除。虽然返回已经过期的状态有利于数据可用性，但不返回过期状态更符合相关数据保护法规的要求。</li>
<li><strong>哪种时间语义被用于定义生存时间？</strong> 在Apache Flink 1.8.0中，用户只能根据处理时间（Processing Time）定义状态生存时间。未来的Flink版本中计划支持事件时间（Event Time）。</li>
</ul>
<p>关于状态生存时间的更多信息，可以参考Flink<a href="https://ci.apache.org/projects/flink/flink-docs-stable/dev/stream/state/state.html#state-time-to-live-ttl">官方文档</a>。</p>
<p>在实现上，状态生存时间特性会额外存储上一次相关状态访问的时间戳。虽然这种方法增加了一些存储开销，但它允许Flink在访问状态、创建检查点、恢复或存储清理过程时可以检查过期状态。</p>
<h3 id="“取出垃圾数据”">“取出垃圾数据”</h3>
<p>在访问状态对象时，Flink将检查其时间戳，并在状态过期时清除状态（是否返回过期状态，则取决于配置的过期数据可见性）。由于这种延迟删除的特性，除非被垃圾回收，否则这些过期数据将仍然占用存储空间。<br>
那么，在没有明确处理过期状态的逻辑情况下，如何删除这些数据呢？通常，我们可以配置不同的策略进行后台删除。</p>
<h4 id="保证完整快照中不包含过期数据">保证完整快照中不包含过期数据</h4>
<p>Flink 1.6.0已经支持在创建检查点（checkpoint）或保存点（savepoint）的完整快照时不包含过期状态。需要注意的是，创建增量快照时并不支持剔除过期状态。完整快照时的过期状态剔除必须如下例所示进行显示启用：</p>
<pre class=" language-java"><code class="prism  language-java">StateTtlConfig ttlConfig <span class="token operator">=</span> StateTtlConfig
    <span class="token punctuation">.</span><span class="token function">newBuilder</span><span class="token punctuation">(</span>Time<span class="token punctuation">.</span><span class="token function">days</span><span class="token punctuation">(</span><span class="token number">7</span><span class="token punctuation">)</span><span class="token punctuation">)</span>
    <span class="token punctuation">.</span><span class="token function">cleanupFullSnapshot</span><span class="token punctuation">(</span><span class="token punctuation">)</span>
    <span class="token punctuation">.</span><span class="token function">build</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
</code></pre>
<p>上述配置并不会影响本地状态存储的大小，但是整个作业的完整快照的大小将会减小。只有当用户从快照重新加载其状态到本地时，才会清除用户的本地状态。</p>
<p>由于上述这些限制，在Flink 1.6.0中程序仍需要过期后主动删除状态。为了改善用户体验，Flink1.8.0引入了两种自主清理策略，分别针对两种状态后端类型：</p>
<h4 id="堆状态后端的增量清理">堆状态后端的增量清理</h4>
<p>此方法只适用于内存堆状态后端（<code>FsStateBackend</code>和<code>MemoryStateBackend</code>）。其基本思路是在存储后端的所有状态条目上维护一个全局的惰性迭代器。某些事件（例如状态访问）会触发增量清理，而每次触发增量清理时，迭代器都会向前迭代删除已遍历的过期数据。以下代码示例展示了如何启用增量清理：</p>
<pre class=" language-java"><code class="prism  language-java">StateTtlConfig ttlConfig <span class="token operator">=</span> StateTtlConfig
    <span class="token punctuation">.</span><span class="token function">newBuilder</span><span class="token punctuation">(</span>Time<span class="token punctuation">.</span><span class="token function">days</span><span class="token punctuation">(</span><span class="token number">7</span><span class="token punctuation">)</span><span class="token punctuation">)</span>
    <span class="token comment">// check 10 keys for every state access</span>
    <span class="token punctuation">.</span><span class="token function">cleanupIncrementally</span><span class="token punctuation">(</span><span class="token number">10</span><span class="token punctuation">,</span> <span class="token boolean">false</span><span class="token punctuation">)</span>
    <span class="token punctuation">.</span><span class="token function">build</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
</code></pre>
<p>如果启用该功能，则每次状态访问都会触发清除。而每次清理时，都会检查一定数量的状态条目是否过期。其中有两个调整参数。第一个定义了每次清理步骤时要检查的状态条目数。第二个参数是一个标志位，用于表示是否在每条记录处理之后（而不仅仅是访问状态），都额外触发清除逻辑。<br>
关于这种方法有两个重要的注意事项：首先是增量清理所花费的时间会增加记录处理的延迟。其次虽然可以忽略不计，但仍然值得一提：如果没有状态被访问或没有记录被处理，过期的状态将不会被删除。</p>
<h4 id="rocksdb状态后端利用后台压缩来清理过期状态">RocksDB状态后端利用后台压缩来清理过期状态</h4>
<p>如果使用RocksDB状态后端，则可以启用另一种清理策略，该策略基于Flink定制的RocksDB压缩过滤器（compaction filter）。RocksDB会定期运行异步的压缩流程以合并数据并减少相关存储的数据量，该定制的压缩过滤器使用生存时间检查状态条目的过期时间戳，并丢弃所有过期值。<br>
激活此功能的第一步，需要设置以下配置选项：<code>state.backend.rocksdb.ttl.compaction.filter.enabled</code>。一旦配置使用RocksDB状态后端后，如以下代码示例将会启用压缩清理策略：</p>
<pre class=" language-java"><code class="prism  language-java">StateTtlConfig ttlConfig <span class="token operator">=</span> StateTtlConfig
    <span class="token punctuation">.</span><span class="token function">newBuilder</span><span class="token punctuation">(</span>Time<span class="token punctuation">.</span><span class="token function">days</span><span class="token punctuation">(</span><span class="token number">7</span><span class="token punctuation">)</span><span class="token punctuation">)</span>
    <span class="token punctuation">.</span><span class="token function">cleanupInRocksdbCompactFilter</span><span class="token punctuation">(</span><span class="token punctuation">)</span>
    <span class="token punctuation">.</span><span class="token function">build</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
</code></pre>
<p>需要注意的是启用Flink的生存时间压缩过滤机制后，会放缓RocksDB的压缩速度。</p>
<h4 id="使用定时器进行状态清理">使用定时器进行状态清理</h4>
<p>另一种手动清除状态的方法是基于Flink的计时器，这是社区目前正在为未来版本评估的一个想法。使用这种方法，将为每个状态访问注册一个清除计时器。这种方法的清理更加精准，因为状态一旦过期就会被立刻删除。但是由于计时器会与原始状态一起存储会消耗空间，开销也更大一些。</p>
<h3 id="未来展望">未来展望</h3>
<p>除了上面提到的基于计时器的清理策略之外，Flink社区还计划进一步改进状态生存时间特性。可能的改进包括为事件时间添加生存时间的支持（目前只支持处理时间）和为可查询状态（queryable state）启用状态生存时间机制。</p>
<h3 id="总结">总结</h3>
<p>基于时间的状态访问限制和应用程序状态大小的控制，是状态流处理领域的常见挑战，Flink的1.8.0版本通过添加对过期状态对象连续性后台清理的支持，显著改进了状态生存时间特性。新的清理机制可以不再需要手动实现状态清理的工作，而且由于惰性清理的机制，执行效率也更高。总得来说，状态生存时间方便用户控制应用程序状态的大小，使得用户可以将精力集中在应用程序的核心逻辑开发上。</p>

