prflr-cpp
=========

PRFLR C++

#ifndef PRFLR_HPP
#define PRFLR_HPP
/*
 *  HOW TO USE
 *
 * // configure profiler
 * // set  profiler server:port  and  set source for timers
 * PRFLR::init("192.168.1.45-testApp", "yourApiKey");
 *
 *
 * //start timer
 * PRFLR::Begin("mongoDB.save");
 *
 * //some code
 * sleep(1);
 *
 * //stop timer
 * PRFLR::End("mongoDB.save");
 *
*/

#include <stdio.h>
#include <stdlib.h>
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

class PRFLRException : public std::exception
{
	std::string reason_;
	public:
	PRFLRException(const std::string &reason):reason_(reason)
	{
	}
	virtual ~PRFLRException() throw()
	{
	}
  virtual const char* what() const throw()
	{
		return reason_.c_str();
	}
};

struct PRFLRPHP
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
};

class PRFLRSender
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
	PRFLRSender(const char *server="prflr.org",int port=4000):delayedSend(false),socket(-1)
	{
		ip = resolve(server,port);
		apikey = "PRFLR-CPP";
	}
	bool bad()
	{
		return ip.sa_family==0;
	}
	virtual ~PRFLRSender() 
	{
		if(socket!=-1) close(socket);
	}
  void Begin(const char *timer)
	{
    timers[timer] = PRFLRPHP::microtime();
  }
	bool End(const char *timer,const char *info = "")
	{
		std::map<string,double>::iterator i = timers.find(timer);
		if(i==timers.end()) return false;
    double time = PRFLRPHP::microtime() - i->second;
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
	void send(const char *timer,double time,const char *info = "")
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
		if(socket==-1) throw PRFLRException("Socket not exist");
		//printf("[%s]\n",message.c_str());
		(void)::sendto(socket,message.c_str(),message.size(),0,&ip,sizeof(ip));
	}
	static struct sockaddr resolve(const char* host,int port)
	{
		sockaddr ret;
		ret.sa_family = 0;
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

class PRFLR
{
	static inline PRFLRSender& sender()
	{
		static PRFLRSender ptrfl_org;
		return ptrfl_org;
	}
	public:
	static void init(const char* source,const char* apikey)
	{
		if(sender().apikey != apikey)
      throw PRFLRException("Unknown apikey.");
		if(sender().bad())
			throw PRFLRException("Unknown sender.");
		sender().socket = ::socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
		if(sender().socket == SOCKET_ERROR) throw PRFLRException("socket");
		///if(!source || strlen(source)==0) ... //source=SERVER_ADDR
		sender().source = source;
		sender().thread = PRFLRPHP::uniqid();
	}
	static inline	void begin(const char* timer)
	{
		sender().Begin(timer);
	}
	static inline	void end(const char* timer, const char *info="")
	{
		sender().End(timer,info);
	}
};

#ifdef MAIN_H
//PRFLRSender PRFLR::sender;
int main()
{
 	PRFLR::init("192.168.1.45-testApp", "PRFLR-CPP");
	PRFLR::begin("mongoDB.save"); //? begin,end upper case
	sleep(1);
	PRFLR::end("mongoDB.save","main|1s");
}
#endif

#endif
