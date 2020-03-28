# ABI revisions

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
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=7af18f8a38ad" target="_blank">20</a></td>
    <td>Add support for compat mode (32-bit exec over 64-bit kernel)</td>
    <td><a href="https://git.evlproject.org/libevl.git/commit/?id=f518a3c" target="_blank">r12</a></td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=a142e2e027dc" target="_blank">19</a></td>
    <td>Make y2038 safe</td>
    <td><a href="https://git.evlproject.org/libevl.git/commit/?id=a9f1e2f" target="_blank">r11</a></td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=b8351b703ffb" target="_blank">18</a></td>
    <td>Plan for supporting a range of ABI revisions</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=87ee9586fa60" target="_blank">17</a></td>
    <td>Replace SIGEVL_ACTION_HOME with RETUSER event</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=231089ed6028" target="_blank">16</a></td>
    <td>Add synchronous breakpoint support</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=6b8a2319c02d" target="_blank">15</a></td>
    <td>Notify stax-related oob exclusion via SIGDEBUG_STAGE_LOCKED</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=3d4ff940c1d3" target="_blank">14</a></td>
    <td>Add stax test helpers to 'hectic' driver</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=a2ba90db409a" target="_blank">13</a></td>
    <td>Add stage exclusion lock mechanism</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=0b5a64ead6f6" target="_blank">12</a></td>
    <td>Add support for recursive gate monitor lock</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=b7c6e2276983" target="_blank">11</a></td>
    <td>Read count of timer expiries as a 64bit value</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=8245a892b9ec" target="_blank">10</a></td>
    <td>Track count of remote thread wakeups</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=f6f6e58cbaff" target="_blank">9</a></td>
    <td>Complete information returned by EVL_THRIOC_GET_STATE</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=9397204d7484" target="_blank">8</a></td>
    <td>Add query for CPU state</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=c1a5ca6a70e7" target="_blank">7</a></td>
    <td>Drop time remainder return from EVL_CLKIOC_SLEEP</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=bc92ac9d3b90" target="_blank">6</a></td>
    <td>Enable fixed-size writes to proxy</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=d9b664b5ecdb" target="_blank">5</a></td>
    <td>Ensure latmus sends the ultimate bulk of results</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=57ce409e23e6" target="_blank">4</a></td>
    <td>Split per-thread debug mode flags</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=c1417f3dbe4f" target="_blank">3</a></td>
    <td>Add count of referrers to poll object shared state</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=3af3b43bdf20" target="_blank">2</a></td>
    <td>Drop obsolete T_MOVED from thread status bits</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=b81555f4f48b" target="_blank">1</a></td>
    <td>Add protocol specifier to monitor element</td>
    <td>-</td>
  </tr>
  <tr>
    <td><a href="https://git.evlproject.org/linux-evl.git/commit/?id=cfab80b242c4" target="_blank">0</a></td>
    <td>Initial revision</td>
    <td>-</td>
  </tr>
</table>

---

{{<lastmodified>}}
