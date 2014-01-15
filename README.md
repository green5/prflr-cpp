// PRFLR CPP

#ifndef PRFLP_HPP
#define PRFLP_HPP

/*
 *  HOW TO USE
 *
 * // configure profiler
 * // set  profiler server:port  and  set source for timers
 * PRFLP::init("192.168.1.45-testApp", "yourApiKey");
 *
 *
 * //start timer
 * PRFLP::begin("mongoDB.save");
 *
 * //some code
 * sleep(1);
 *
 * //stop timer
 * PRFLP::end("mongoDB.save");
 *
 * //test: g++ -Wall -DMAIN_H -x c++ prflr.hpp && ./a.out
*/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <netdb.h>
#include <sys/socket.h>
#include <sys/timeb.h> 

#ifdef _WIN32
#include <inaddr.h>
#endif
#ifndef SOCKET_ERROR
#define SOCKET_ERROR -1
#endif

#include <string>
#include <map>
#include <exception>
#include <sstream>

class PRFLPException : public std::exception
{
	std::string reason_;
	public:
	PRFLPException(const std::string &reason):reason_(reason)
	{
	}
	virtual ~PRFLPException() throw()
	{
	}
	virtual const char* what() const throw()
	{
		return reason_.c_str();
	}
};

#ifdef __linux
#include <syscall.h>
#else
#include <pthread.h>
#endif

struct PRFLPMisc
{
	static std::string uniqid()
	{
		struct timeb t;
		ftime(&t);
		std::ostringstream ret;
		ret << t.time << t.millitm; 
		return ret.str(); 
	}
	static double microtime()
	{
		struct timeb t;
		ftime(&t);
		return 1000.*t.millitm + t.time;
	}
	static std::string threadid(const char* suffix="")
	{
		char ret[20];
#ifdef __linux
		pid_t x = (pid_t)syscall(SYS_gettid);
		snprintf(ret,sizeof(ret),"%d%s",(int)x,suffix);
#else
		ulong x = (ulong)pthread_self();
		snprintf(ret,sizeof(ret),"%lx%s",x,suffix);
#endif
		return ret;
	}
};

class PRFLPSender
{
	typedef std::string string;
	std::map<string,double> timers;
	int delayedSend;
	sockaddr ip;
	public:
	int socket;
	string source;
	string thread;
	string apikey;
	public:
	PRFLPSender(const char *server="prflr.org",int port=4000):delayedSend(false),socket(-1)
	{
		ip = resolve(server,port);
		apikey = "PRFLP-CPP";
	}
	bool bad()
	{
		return ip.sa_family==0;
	}
	virtual ~PRFLPSender() 
	{
		if(socket!=-1) close(socket);
	}
	void Begin(const string& timer)
	{
		timers[timer] = PRFLPMisc::microtime();
	}
	bool End(const string& timer,const char *info = "")
	{
		std::map<string,double>::iterator i = timers.find(timer);
		if(i==timers.end()) return false;
		double time = PRFLPMisc::microtime() - i->second;
		send(timer, time, info);
		timers.erase(i);
		return true;
	}
	private:
	static string substr(const string &a,size_t n,char c='|')
	{
		string ret;
		string::const_iterator end = a.end();
		for(string::const_iterator i = a.begin();i!=end;++i)
		{
			if(*i!='|') ret += *i;
		}
		ret = ret.substr(0,n);
		if(c) ret += '|';
		return ret;
	}
	void send(const string& timer,double time,const char *info = "")
	{
		char ntime[100];
		snprintf(ntime,sizeof(ntime),"%.3f",time);
		string message = 
			substr(thread,32)
			+ substr(source,32)
			+ substr(timer,48)
			+ substr(ntime,16)
			+ substr(info,32)
			+ substr(apikey,32,0);
		if(socket==-1) throw PRFLPException("Socket not exist");
#ifdef MAIN_H
		printf("[%s]\n",message.c_str());
#else
		(void)::sendto(socket,message.c_str(),message.size(),0,&ip,sizeof(ip));
#endif
	}
	static struct sockaddr resolve(const char* host,int port)
	{
		sockaddr ret;
		memset(&ret,0,sizeof(ret));
		if(1==1)
		{
			sockaddr_in t;
			int a,b,c,d;
			if(isdigit(host[0]) && sscanf(host, "%d.%d.%d.%d", &a,&b,&c,&d)==4)
			{
				t.sin_addr.s_addr = htonl((a<<24)+(b<<16)+(c<<8)+d);
			}
			else
			{
				struct hostent *h = gethostbyname(host);
				if(h==NULL) return ret;
				t.sin_addr.s_addr = *(u_long*) h->h_addr;
			}
			t.sin_family = AF_INET;
			t.sin_port = htons(port);
			ret = *(sockaddr*)&t;
		}
		// TODO: add INET6
		return ret;
	}
};

class PRFLP
{
	static inline PRFLPSender& sender()
	{
		static PRFLPSender ptrfl_org;
		return ptrfl_org;
	}
	public:
	static void init(const char* source,const char* apikey)
	{
		if(sender().apikey != apikey)
			throw PRFLPException("Unknown apikey.");
		if(sender().bad())
			throw PRFLPException("Unknown sender.");
		sender().socket = ::socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
		if(sender().socket == SOCKET_ERROR) throw PRFLPException("socket");
		///if(!source || strlen(source)==0) ... //source=SERVER_ADDR
		sender().source = source;
		sender().thread = PRFLPMisc::uniqid();
	}
	static inline void begin(const char* timer)
	{
		sender().Begin(PRFLPMisc::threadid("-") + timer);
	}
	static inline void end(const char* timer, const char *info="")
	{
		sender().End(PRFLPMisc::threadid("-") + timer,info);
	}
};

#ifdef MAIN_H
int main()
{
 	PRFLP::init("192.168.1.45-testApp", "PRFLP-CPP");
	PRFLP::begin("mongoDB.save");
	sleep(1);
	PRFLP::end("mongoDB.save","main|1s");
}
#endif

#endif
