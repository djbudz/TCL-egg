##
#
# ipv6dns.tcl v1.4 - by strikelight ([sL] @ EFNet) (09/25/03)
#
# For eggdrop1.1.5-eggdrop1.6.x
#
# Requires: nslookup tool installed on shell server
#         : bgexec.tcl (available from www.TCLScript.com) to be loaded
#         : BEFORE this script
#
# Contact:
# - E-Mail: strikelight@tclscript.com
# - WWW   : http://www.TCLScript.com
# - IRC   : #Scripting @ EFNet
#
#
# Usage:
#
# - Edit the configuration below
#
# - In channel type: !ipv6dns <IPv6-host/ip)
###

## CONFIGURATION ##

# Public command to use to perform ipv6 lookups
set ipv6dns(pubcmd) "!dns6"

# Flag required to use public trigger
# set ipv6dns(pubflag) "o|o"
set ipv6dns(pubflag) "-"

# DCC command to use to perform ipv6 lookups
set ipv6dns(dcccmd) "dns6"

# Flag required to use dcc command
set ipv6dns(dccflag) "o|o"

# Uncomment and set this variable only if script says it can't locate the nslookup tool
# set ipv6dns(nslookup) "/path/to/nslookup"

## END OF CONFIGURATION ##

if {![info exists bgexec(version)]} {
  putlog "Error: bgexec.tcl by strikelight not found."
  putlog "Error: bgexec.tcl is required for ipv6dns.tcl to function properly."
  putlog "Error: You may get bgexec.tcl from http://www.TCLScript.com/"
}

if {[catch {exec nslookup 127.0.0.1}] && [catch {exec nslookup localhost}]} {
  if {[info exists ipv6dns(nslookup)] && ![file exists $ipv6dns(nslookup)]} {
    putlog "Error: ipv6dns(nslookup) variable is set, but is incorrect. ipv6dns.tcl will not function."
  } elseif {![info exists ipv6dns(nslookup)] && [catch {set ipv6dns(nslookup) [exec which nslookup]}]} {
    putlog "Error: nslookup tool does not appear to be installed on this system."
    putlog "Error: nslookup is required for ipv6dns.tcl to function properly."
    putlog "Error: if nslookup IS installed on the system, edit this script, "
    putlog "Error: and change the ipv6dns(nslookup) variable under configuration."
  }
}

bind pub $ipv6dns(pubflag) $ipv6dns(pubcmd) ipv6_pub_dns
bind dcc $ipv6dns(dccflag) $ipv6dns(dcccmd) ipv6_dcc_dns

proc ipv6tonibble {ip {type "int"}} {
  set newip ""
  set nlist ""
  set mlist [split $ip ":"]
  foreach element $mlist {
    if {[string trim $element] == ""} {
      set totzeros [expr 8 - [llength $mlist]]
      for {set i 0} {$i <= $totzeros} {incr i} {
        lappend nlist 0
      }
    } else {
      lappend nlist $element
    }
  }
  for {set i [expr [llength $nlist] - 1]} {$i >= 0} {incr i -1} {
    set seg [lindex $nlist $i]
    for {set j 0} {$j < 4} {incr j} {
      if {$seg != ""} {
        append newip "[string index $seg [expr [string length $seg] - 1]]."
        if {[string length $seg] > 1} {
          set seg [string range $seg 0 [expr [string length $seg] - 2]]
        } else {
          set seg ""
        }
      } else {
        append newip "0."
      }
    }
  }
  append newip "ip6.$type"
  return $newip
}

proc ipv6_remove_queue {who} {
  global ipv6dns
  set dloc [lsearch $ipv6dns(queue) $who]
  if {$dloc != -1} {set ipv6dns(queue) [lreplace $ipv6dns(queue) $dloc $dloc]}
}

proc ipv6_callback {type where who orig ip6type input} {
  global botnick ipv6dns
  set result ""
  if {![info exists ipv6dns(nslookup)]} {
    set cmd "nslookup"
  } else {
    set cmd $ipv6dns(nslookup)
  }
  set fnd 0
  foreach line [split $input "\n"] {
    if {[string match "*ip6.\[int|arpa\]*name*=*" $line] || [string match "*IPv6 address*=*" $line]} {
      set result [string trim [lindex [split $line "="] 1]]
      break
    } elseif {[string match "*has AAAA*" $line]} {
      set result [lindex [split $line] [expr [llength [split $line]] - 1]]
      break
    } elseif {[string match "*Name:*" $line]} {
      set name [lindex [split $line] [expr [llength [split $line]] - 1]]
      if {[string tolower $name] == [string tolower $orig]} {
        set fnd 1
      }
    } elseif {($fnd) && ([string match "*Address:*" $line])} {
      set result [lindex [split $line] [expr [llength [split $line]] - 1]]
      break
    }
  }
  if {($result == "") && ($ip6type == 1) && ([string match "*:*" $orig])} {
    set lookup [ipv6tonibble $orig arpa]
    if {[catch {bgexec "$cmd -type=any $lookup" [list ipv6_callback $type $where $who $orig 2]} err]} {
      set result ""
    } else {
      return
    }
  }
  if {($result != "") && ([string index $result [expr [string length $result] - 1]] == ".")} {
    set result [string range $result 0 [expr [string length $result] - 2]]
  }
  if {$result == ""} { set result "Error during lookup, or lookup impossible." }
  if {$type == "dcc"} {
    if {[valididx $where]} {
      putidx $where "\[\002ipv6\002] $orig = $result"
      ipv6_remove_queue $who
      return
    }
  } elseif {$type == "pub"} {
    if {[onchan $botnick $where]} {
      puthelp "PRIVMSG $where :\[\002ipv6\002] \037$who\037: $orig = $result"
      ipv6_remove_queue $who
      return
    }
  }
}

proc ipv6_pub_dns {nick uhost hand chan text} {
  global ipv6dns
  if {$hand == "*"} {
    set who [string tolower $nick]
  } else {
    set who [string tolower $hand]
  }
  if {![info exists ipv6dns(nslookup)]} {
    set cmd "nslookup"
  } else {
    set cmd $ipv6dns(nslookup)
  }
  if {[lsearch $ipv6dns(queue) $who] != -1} {
    puthelp "NOTICE $nick :You already have a request in queue, please wait."
    return
  }
  set orig [lindex $text 0]
  if {$orig == ""} {
    puthelp "NOTICE $nick :Usage: $ipv6dns(pubcmd) <host/ipv6>"
    return
  }
  regsub -all {[^a-zA-Z0-9:.\-]} $orig "" orig
  if {![string match "*\[:.\]*" $orig]} {
    puthelp "NOTICE $nick :$orig does not appear to be a valid host/ipv6 address."
    return
  }
  if {[string match "*:*" $orig]} {
    set lookup [ipv6tonibble $orig]
    set atype "any"
    set ptype "1"
  } else {
    set lookup $orig
    set atype "AAAA"
    set ptype "0"
  }
  if {[catch {bgexec "$cmd -type=$atype $lookup" [list ipv6_callback pub $chan $who $orig $ptype]} err]} {
    puthelp "PRIVMSG $chan :Error during execution of nslookup."
    return
  }
  puthelp "NOTICE $nick :Processing your request, one moment..."
  lappend ipv6dns(queue) $who
  utimer 15 [list ipv6_remove_queue $who]
}

proc ipv6_dcc_dns {hand idx text} {
  global ipv6dns
  set who [string tolower $hand]
  if {![info exists ipv6dns(nslookup)]} {
    set cmd "nslookup"
  } else {
    set cmd $ipv6dns(nslookup)
  }
  if {[lsearch $ipv6dns(queue) $who] != -1} {
    putidx $idx "You already have a request in queue, please wait."
    return 0
  }
  set orig [lindex $text 0]
  if {$orig == ""} {
    putidx $idx "Usage: $ipv6dns(dcccmd) <host/ipv6>"
    return 0
  }
  regsub -all {[^a-zA-Z0-9:.\-]} $orig "" orig
  if {![string match "*\[:.\]*" $orig]} {
    putidx $idx "$orig does not appear to be a valid host/ipv6 address."
    return 1
  }
  if {[string match "*:*" $orig]} {
    set lookup [ipv6tonibble $orig]
    set ptype "1"
  } else {
    set lookup $orig
    set ptype "0"
  }
  if {[catch {bgexec "$cmd -type=any $lookup" [list ipv6_callback dcc $idx $who $orig $ptype]} err]} {
    putidx $idx "Error during execution of nslookup."
    return
  }
  putidx $idx "Processing your request, one moment..."
  lappend ipv6dns(queue) $who
  utimer 15 [list ipv6_remove_queue $who]
  return 1
}

if {![info exists ipv6dns(queue)]} { set ipv6dns(queue) "" }
set ipv6dns(version) "1.4"

putlog "ipv6dns.tcl v$ipv6dns(version) by strikelight now loaded."
