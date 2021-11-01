---
title: ABI revisions
---

<div>
<style>
#abimap {
       width: 35%;
       margin-left: auto;
       margin-right: auto;
}
#abimap th {
       text-align: center;
}
#abimap td {
       text-align: center;
}
#abimap tr:nth-child(even) {
       background-color: #f2f2f2;
}
</style>

<table id="abimap">
  <col width="5%">
  <col width="90%">
  <col width="5%">
  <tr>
    <th>Revision</th>
    <th>Purpose</th> 
    <th>libevl release</th>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/fcc7a8c49a4e5a632f60fd8da5a84855c618df04" target="_blank">27</a></td>
    <td>Handle prctl()-based syscall form. This enables Valgrind for EVL applications, while keeping backward compatibility for the legacy call form.
    <td><a href="https://git.xenomai.org/xenomai4/libevl/-/tags/r28" target="_blank">r28</a></td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/41dc88ab524cd32878646d98b57d390e2bcc1198" target="_blank">26</a></td>
    <td>Add socket interface.
    <td><a href="https://git.xenomai.org/xenomai4/libevl/-/tags/r26" target="_blank">r21</a></td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/a8aa18f663feaabde73285bad3b000e02e0c2332" target="_blank">25</a></td>
    <td>Add latmus request for measuring in-band response time to synthetic interrupt latency.
    <td><a href="https://git.xenomai.org/xenomai4/libevl/-/tags/r21" target="_blank">r21</a></td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/5be4fa6a6900e21" target="_blank">24</a></td>
    <td>Add proxy read side.
    <td><a href="https://git.xenomai.org/xenomai4/libevl/-/tags/r19" target="_blank">r19</a></td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/8d05a80bc0d5a94" target="_blank">23</a></td>
    <td>Add the Observable element, and thread observability.
    <td><a href="https://git.xenomai.org/xenomai4/libevl/-/tags/r17" target="_blank">r17</a></td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/e19190719052a60" target="_blank">22</a></td>
    <td>Add EVL_THRIOC_UNBLOCK, EVL_THRIOC_DEMOTE and EVL_THRIOC_YIELD, update EVL_THRIOC_*_MODE operations.
    <td><a href="https://git.xenomai.org/xenomai4/libevl/-/tags/r16" target="_blank">r16</a></td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/a2ee2abf2986188" target="_blank">21</a></td>
    <td>Introduce element visibility attribute</td>
    <td><a href="https://git.xenomai.org/xenomai4/libevl/-/tags/r15" target="_blank">r15</a></td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/7af18f8a38ad68f581cf770819ccb98c29f41586" target="_blank">20</a></td>
    <td>Add support for compat mode (32-bit exec over 64-bit kernel)</td>
    <td><a href="https://git.xenomai.org/xenomai4/libevl/-/tags/r12" target="_blank">r12</a></td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/a142e2e027dc" target="_blank">19</a></td>
    <td>Make y2038 safe</td>
    <td><a href="https://git.xenomai.org/xenomai4/libevl/-/tags/r11" target="_blank">r11</a></td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/b8351b703ffb" target="_blank">18</a></td>
    <td>Plan for supporting a range of ABI revisions</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/87ee9586fa60" target="_blank">17</a></td>
    <td>Replace SIGEVL_ACTION_HOME with RETUSER event</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/231089ed6028" target="_blank">16</a></td>
    <td>Add synchronous breakpoint support</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/6b8a2319c02d" target="_blank">15</a></td>
    <td>Notify stax-related oob exclusion via SIGDEBUG_STAGE_LOCKED</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/3d4ff940c1d3" target="_blank">14</a></td>
    <td>Add stax test helpers to 'hectic' driver</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/a2ba90db409a" target="_blank">13</a></td>
    <td>Add stage exclusion lock mechanism</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/0b5a64ead6f6" target="_blank">12</a></td>
    <td>Add support for recursive gate monitor lock</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/b7c6e2276983" target="_blank">11</a></td>
    <td>Read count of timer expiries as a 64bit value</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/8245a892b9ec" target="_blank">10</a></td>
    <td>Track count of remote thread wakeups</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/f6f6e58cbaff" target="_blank">9</a></td>
    <td>Complete information returned by EVL_THRIOC_GET_STATE</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/9397204d7484" target="_blank">8</a></td>
    <td>Add query for CPU state</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/c1a5ca6a70e7" target="_blank">7</a></td>
    <td>Drop time remainder return from EVL_CLKIOC_SLEEP</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/bc92ac9d3b90" target="_blank">6</a></td>
    <td>Enable fixed-size writes to proxy</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/d9b664b5ecdb" target="_blank">5</a></td>
    <td>Ensure latmus sends the ultimate bulk of results</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/57ce409e23e6" target="_blank">4</a></td>
    <td>Split per-thread debug mode flags</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/c1417f3dbe4f" target="_blank">3</a></td>
    <td>Add count of referrers to poll object shared state</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/3af3b43bdf20" target="_blank">2</a></td>
    <td>Drop obsolete T_MOVED from thread status bits</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/b81555f4f48b" target="_blank">1</a></td>
    <td>Add protocol specifier to monitor element</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.xenomai.org/xenomai4/linux-evl/-/commit/cfab80b242c4" target="_blank">0</a></td>
    <td>Initial revision</td>
    <td>-</td>
  </tr>
</table>
</div>

---

{{<lastmodified>}}
