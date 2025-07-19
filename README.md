# Kunkin KP184 RS232 control program for Linux, and (maybe) Windows

**Disclaimer: The owner of this GitHub account is NOT the author of this program.
It is hosted here with the author's permission, for the sole benefit of the community, without any support of any kind.**

Feel free to use Pull requests or Issues to share remarks, but don't expect too hardly an answer from the author.

All the content is in the file kp184.c

Original readme by the author :

    This program, called kp184, is meant to facilitate communication with Kunkin KP184 electronic load
    using serial port (RS232). It can be used from any (script) language you are familiar with.
    Copyright (c) 2025 rat de combat - first published on forum.hardware.fr
    This program is free software: you can redistribute it and/or modify it under the
    terms of the GNU General Public License as published by the Free Software Foundation,
    either version 3 of the License, or (at your option) any later version.
    This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY;
    without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
    See the GNU General Public License for more details.
    You should have received a copy of the GNU General Public License along with this program.
    If not, see <https://www.gnu.org/licenses/>.
    
    PROGRAM: kp184 Copyright (c) 2025 rat de combat - first published on forum.hardware.fr
    DEPENDENCY: libserialport, hosted by Sigrok project at https://sigrok.org/ - see there for compiling it
    SUPPORTED OS: Linux (tested), Windows (untested), maybe others (untested) depending on libserialport
    COMPILATION (Linux, assuming libserialport.h and libserialport.so are in the same directory, adjust as needed): gcc -Wall -Wextra -Werror -I. -L. -Wl,-rpath=. -o kp184 kp184.c -lserialport
    USE: kp184 COMMAND [ARGUMENTS]
    kp184 v                            -> show _v_ersion and copyright on stdout
    kp184 i SERIAL_PORT BAUDRATE NODE  -> _i_nit, to be called first once, creates a small internal config file in current directory
    kp184 c                            -> _c_leanup, to be called at end of script, removes the file created above (file can be removed manually too)
    kp184 s on|off                     -> _s_witch load on/off
    kp184 m MODE VALUE                 -> change _m_ode to MODE (v|c|r|p) and set voltage/current/power/resistance to VALUE (double)
    kp184 r                            -> _r_ead mode and real voltage and current and print on stdout
    Always check return value, will be non-zero on error. See #define at the beginning of code.
    The firmware of the KP184 does not seem very good/stable and the documentation is really sparse (and bad quality), so YMMV using this program. Add plenty of checks to your own scripts. Never run the load without your physical presence/surveillance.
