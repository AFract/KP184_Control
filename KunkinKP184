    /*
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
    */
    /*
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
    */
    #include <stdlib.h>
    #include <stdio.h>
    #include <stdint.h>
    #include <stdbool.h>
    #include <string.h>
    #include <libserialport.h>
    #ifdef WIN32
    #include <windows.h>
    #else
    #include <time.h>
    #endif
    //return values
    #define ERR_NO_ERROR 0
    #define ERR_NEED_ARGUMENTS 1
    #define ERR_INVALID_COMMAND 2
    #define ERR_INVALID_ARGUMENT 3
    #define ERR_FOPEN_CONF_FILE 4
    #define ERR_FWRITE_CONF_FILE 5
    #define ERR_FREAD_CONF_FILE 6
    #define ERR_REMOVE_CONF_FILE 7
    #define ERR_LIBSERIALPORT_ERROR 8
    #define ERR_INVALID_RESPONSE 9
    #define ERR_INVALID_CRC 10
    #define NAME_CONFFILE "kp184_conf"
    #define SIZE_NAME_SERIAL_PORT 50
    #define DEBUG_SHOW_BYTES 0
    #define DELAY_MS_BETWEEN_COMMANDS 200 //minimum value?
    #define TOOL_VERSION "0.1"
    typedef struct
    {
        char port_name[SIZE_NAME_SERIAL_PORT+1];
        uint32_t baudrate;
        uint8_t node;
    } config_file_t;
    //do not change
    #define CMD_READ_SINGLE_REG 0x03
    #define CMD_WRITE_SINGLE_REG 0x06
    #define REG_LOAD_ON_OFF 0x010E
    #define REG_LOAD_MODE 0x0110
    #define REG_CV_SETTING 0x0112
    #define REG_CC_SETTING 0x0116
    #define REG_CR_SETTING 0x011A
    #define REG_CW_SETTING 0x011E
    #define REG_U_MEASURE 0x0122
    #define REG_I_MEASURE 0x0126
    #define MODE_CV 0x00
    #define MODE_CC 0x01
    #define MODE_CR 0x02
    #define MODE_CP 0x03
    static struct sp_port * port=NULL;
    static void check_abort(const enum sp_return ret)
    {
        if(ret<0)
        {
            fprintf(stderr, "kp184: something went wrong with libserialport (code %d)\n", ret);
            exit(ERR_LIBSERIALPORT_ERROR);
        }
    }
    static void close_free_port(void)
    {
        if(port!=NULL)
        {
            check_abort(sp_close(port));
            sp_free_port(port);
        }
    }
    static void wait_ms(const uint_fast32_t ms)
    {
    #ifdef WIN32
        Sleep(ms);
    #else
        struct timespec ts;
        ts.tv_sec=ms/1000;
        ts.tv_nsec=(ms%1000)*1000000;
        nanosleep(&ts, NULL);
    #endif
    }
    static uint16_t compute_crc(uint8_t const * ptr, uint_fast8_t len_data)
    {
        uint_fast8_t i;
        uint16_t crc=0xFFFF;
        while(len_data--)
        {
            crc^=(*ptr);
            for(i=0; i<8; i++)
            {
                if(crc&1)
                {
                    crc>>=1;
                    crc^=0xA001;
                }
                else
                    crc>>=1;
            }
            ptr++;
        }
        return crc;
    }
    static void set_crc(uint8_t * buffer, uint_fast8_t len_data)
    {
        uint16_t crc=compute_crc(buffer, len_data);
        buffer[len_data++]=(crc>>8)&0xFF;
        buffer[len_data]=crc&0xFF;
    }
    static bool is_good_crc(uint8_t const * buffer, uint_fast8_t len_total)
    {
        uint16_t computed_crc=compute_crc(buffer, len_total-2);
        uint16_t real_crc=((uint16_t)(buffer[len_total-2])<<8)|buffer[len_total-1];
        return computed_crc==real_crc;
    }
    static config_file_t read_conf_file(void)
    {
        FILE * f=fopen(NAME_CONFFILE, "rb" );
        if(f==NULL)
        {
            fprintf(stderr, "kp184: read_conf_file: could not open internal config file %s\n", NAME_CONFFILE);
            exit(ERR_FOPEN_CONF_FILE);
        }
        config_file_t conf;
        if(fread(&conf, sizeof(config_file_t), 1, f)!=1)
        {
            fprintf(stderr, "kp184: read_conf_file: could not read from internal config file %s\n", NAME_CONFFILE);
            exit(ERR_FREAD_CONF_FILE);
        }
        fclose(f);
        return conf;
    }
    static void open_port(void)
    {
        if(port!=NULL)
            return;
       
        config_file_t conf=read_conf_file();
       
        check_abort(sp_get_port_by_name(conf.port_name, &port));
        check_abort(sp_open(port, SP_MODE_READ_WRITE));
    }
    static void prepare_single_reg_write(uint8_t * buffer, const uint32_t reg_addr, const uint32_t value, uint8_t * response)
    {
        config_file_t conf=read_conf_file();
        uint8_t * ptr=buffer;
        *(ptr++)=conf.node;
        *(ptr++)=CMD_WRITE_SINGLE_REG;
        *(ptr++)=(reg_addr>>8)&0xFF;
        *(ptr++)=reg_addr&0xFF;
        *(ptr++)=0x00;
        *(ptr++)=0x01;
        *(ptr++)=0x04;
        *(ptr++)=(value>>24)&0xFF;
        *(ptr++)=(value>>16)&0xFF;
        *(ptr++)=(value>>8)&0xFF;
        *(ptr++)=value&0xFF;
        set_crc(buffer, 11);
        uint8_t * ptr_response=response;
        *(ptr_response++)=conf.node;
        *(ptr_response++)=CMD_WRITE_SINGLE_REG;
        *(ptr_response++)=(reg_addr>>8)&0xFF;
        *(ptr_response++)=reg_addr&0xFF;
        *(ptr_response++)=0x00;
        *(ptr_response++)=0x01;
        *(ptr_response++)=0x04;
        set_crc(response, 7);
    }
    static void prepare_single_reg_read(uint8_t * buffer, const uint32_t reg_addr)
    {
        config_file_t conf=read_conf_file();
        uint8_t * ptr=buffer;
        *(ptr++)=conf.node;
        *(ptr++)=CMD_READ_SINGLE_REG;
        *(ptr++)=(reg_addr>>8)&0xFF;
        *(ptr++)=reg_addr&0xFF;
        *(ptr++)=0x00;
        *(ptr++)=0x04;
        set_crc(buffer, 6);
    }
    static void send_data(uint8_t const * buffer, uint_fast8_t len)
    {
        open_port();
    #if DEBUG_SHOW_BYTES
        uint_fast8_t i;
        printf("transmitting %u bytes: ", len);
        for(i=0; i<len; i++)
            printf("%02X ", buffer[i]);
        printf("\n" );
    #endif
        check_abort(sp_blocking_write(port, buffer, len, 100));
        check_abort(sp_drain(port));
    }
    static uint_fast8_t receive_data(uint8_t * const buffer, uint_fast8_t len_max)
    {
        open_port();
        enum sp_return ret=sp_blocking_read(port, buffer, len_max, 250);
        check_abort(ret);
        if((uint)ret!=len_max)
        {
            fprintf(stderr, "kp184: expected %u bytes but received %u bytes\n", len_max, (uint)ret);
            exit(ERR_INVALID_RESPONSE);
        }
    #if DEBUG_SHOW_BYTES
        uint_fast8_t i;
        printf("received %d bytes: ", ret);
        for(i=0; i<ret; i++)
            printf("%02X ", buffer[i]);
        printf("\n" );
    #endif
        if(!is_good_crc(buffer, ret))
        {
            fprintf(stderr, "kp184: invalid CRC in response\n" );
            exit(ERR_INVALID_CRC);
        }
        return ret;
    }
    static void check_response(uint8_t const * const real, uint8_t const * const expected, uint_fast8_t len)
    {
        if(memcmp(real, expected, len))
        {
            fprintf(stderr, "kp184: invalid response\n" );
            exit(ERR_INVALID_RESPONSE);
        }
    }
    static void do_init(int nb, char ** args)
    {
        if(nb!=3)
        {
            fprintf(stderr, "kp184: init: missing or too many argument(s)\n" );
            exit(ERR_NEED_ARGUMENTS);
        }
       
        char * port_name=args[0];
        uint_fast32_t baudrate=atoi(args[1]);
        uint8_t node=atoi(args[2]);
        struct sp_port *port;
        check_abort(sp_get_port_by_name(port_name, &port));
        check_abort(sp_open(port, SP_MODE_READ_WRITE));
        check_abort(sp_set_baudrate(port, baudrate));
        check_abort(sp_set_bits(port, 8));
        check_abort(sp_set_parity(port, SP_PARITY_NONE));
        check_abort(sp_set_stopbits(port, 1));
        check_abort(sp_set_flowcontrol(port, SP_FLOWCONTROL_NONE));
        config_file_t conf;
        strncpy(conf.port_name, port_name, SIZE_NAME_SERIAL_PORT);
        conf.port_name[SIZE_NAME_SERIAL_PORT]='\0';
        conf.baudrate=baudrate;
        conf.node=node;
        FILE * f=fopen(NAME_CONFFILE, "wb" );
        if(f==NULL)
        {
            fprintf(stderr, "kp184: init: could not create internal config file %s\n", NAME_CONFFILE);
            exit(ERR_FOPEN_CONF_FILE);
        }
        if(fwrite(&conf, sizeof(config_file_t), 1, f)!=1)
        {
            fprintf(stderr, "kp184: init: could not write to internal config file %s\n", NAME_CONFFILE);
            exit(ERR_FWRITE_CONF_FILE);
        }
        fclose(f);
    }
    static void do_cleanup(void)
    {
        if(remove(NAME_CONFFILE)<0)
        {
            fprintf(stderr, "kp184: cleanup: failed to remove internal config file %s\n", NAME_CONFFILE);
            exit(ERR_REMOVE_CONF_FILE);
        }
    }
    static void do_switch(int nb, char ** args)
    {
        if(nb!=1)
        {
            fprintf(stderr, "kp184: switch: missing or too many argument(s)\n" );
            exit(ERR_NEED_ARGUMENTS);
        }
        char * state=args[0];
        uint32_t val;
        if(!strcmp(state, "on" ))
            val=1;
        else if(!strcmp(state, "off" ))
            val=0;
        else
        {
            fprintf(stderr, "kp184: switch: invalid argument %s\n", state);
            exit(ERR_INVALID_ARGUMENT);
        }
       
        uint8_t cmd[13];
        uint8_t expected_response[9];
        prepare_single_reg_write(cmd, REG_LOAD_ON_OFF, val, expected_response);
        send_data(cmd, 13);
        uint8_t response[9];
        receive_data(response, 9);
        check_response(response, expected_response, 9);
    }
    //CAUTION: we might get rounding errors with double, we really should avoid floating point here...
    static void do_set_mode(int nb, char ** args)
    {
        if(nb!=2)
        {
            fprintf(stderr, "kp184: mode: missing or too many argument(s)\n" );
            exit(ERR_NEED_ARGUMENTS);
        }
        char mode=args[0][0];
        uint32_t mode_bin;
        uint32_t reg_value;
        double value_double=atof(args[1]);
        uint32_t value_u32;
        switch(mode)
        {
            case 'v': mode_bin=MODE_CV; reg_value=REG_CV_SETTING; value_u32=(uint32_t)(value_double*1000); break;
            case 'c': mode_bin=MODE_CC; reg_value=REG_CC_SETTING; value_u32=(uint32_t)(value_double*1000); break;
            case 'r': mode_bin=MODE_CR; reg_value=REG_CR_SETTING; value_u32=(uint32_t)(value_double*10); break;
            case 'p': mode_bin=MODE_CP; reg_value=REG_CW_SETTING; value_u32=(uint32_t)(value_double*100); break;
            default:
                fprintf(stderr, "kp184: mode: invalid mode '%c'\n", mode);
                exit(ERR_INVALID_ARGUMENT);
                break;
        }
        uint8_t cmd[13];
        uint8_t expected_response[9];
        prepare_single_reg_write(cmd, REG_LOAD_MODE, mode_bin, expected_response);
        send_data(cmd, 13);
        uint8_t response[9];
        receive_data(response, 9);
        check_response(response, expected_response, 9);
        wait_ms(DELAY_MS_BETWEEN_COMMANDS);
       
        prepare_single_reg_write(cmd, reg_value, value_u32, expected_response);
        send_data(cmd, 13);
        receive_data(response, 9);
        check_response(response, expected_response, 9);
    }
    static void do_read_mode_and_value(void)
    {
        uint8_t cmd[8];
        prepare_single_reg_read(cmd, REG_LOAD_MODE);
        send_data(cmd, 8);
        uint8_t response[9];
        receive_data(response, 9);
        uint8_t mode_bin=response[6];
        if(mode_bin>3)
        {
            fprintf(stderr, "kp184: invalid mode\n" );
            exit(ERR_INVALID_RESPONSE);
        }
        const char mode[4]={'V', 'C', 'R', 'P'};
        printf("MODE: C%c - ", mode[mode_bin]);
        wait_ms(DELAY_MS_BETWEEN_COMMANDS);
        prepare_single_reg_read(cmd, REG_U_MEASURE);
        send_data(cmd, 8);
        receive_data(response, 9);
        uint32_t raw_voltage=(((uint32_t)response[3])<<24)|(response[4]<<16)|(response[5]<<8)|response[6];
        printf("REAL_VOLTAGE: %06.3fV - ", (float)raw_voltage/1000);
        wait_ms(DELAY_MS_BETWEEN_COMMANDS);
        prepare_single_reg_read(cmd, REG_I_MEASURE);
        send_data(cmd, 8);
        receive_data(response, 9);
        uint32_t raw_current=(((uint32_t)response[3])<<24)|(response[4]<<16)|(response[5]<<8)|response[6];
        printf("REAL_CURRENT: %06.3fA\n", (float)raw_current/1000);
    }
    int main(int argc, char ** argv)
    {
        if(argc<2)
        {
            fprintf(stderr, "kp184: wrong usage: kp184 COMMAND [ARGUMENTS]\n" );
            exit(ERR_NEED_ARGUMENTS);
        }
        atexit(&close_free_port);
        char cmd=argv[1][0];
        switch(cmd)
        {
            case 'v':
                printf("kp184 version %s - Copyright (c) 2025 rat de combat\n", TOOL_VERSION);
                break;
       
            case 'i':
                do_init(argc-2, &argv[2]);
                printf("OK\n" );
                break;
            case 'c':
                do_cleanup();
                printf("OK\n" );
                break;
           
            case 's':
                do_switch(argc-2, &argv[2]);
                printf("OK\n" );
                break;
            case 'm':
                do_set_mode(argc-2, &argv[2]);
                printf("OK\n" );
                break;
            case 'r':
                do_read_mode_and_value();
                break;
            default:
                fprintf(stderr, "kp184: invalid command '%c'\n", cmd);
                exit(ERR_INVALID_COMMAND);
                break;
        }
       
        return ERR_NO_ERROR;
    }
