use strict;
use warnings;
use threads;
use threads::shared;
use IO::Socket::SSL;
use Fcntl qw/:flock/;


my $thr_cnt = 15;
my @cat_list : shared = qw/ARCADE BRAIN CARDS CASUAL GAME_WALLPAPER RACING SPORTS_GAMES GAME_WIDGETS/;
my $page_limit = 15;
my $res_file = 'result2.txt';

$| = 1;
my @trl = ();
my $w_lock : shared;

my $time = time;

$trl[$_] = threads->create(\&main) for 0..$thr_cnt - 1;
$_->join for @trl;

print 'Time elapsed: ', time - $time, $/;

sub main
{
    while(1)
    {
        my $cat_name = shift @cat_list or last;
        
        for(my $i = 0; $i < $page_limit; $i++)
        {
            print $cat_name, ' - ', $i, $/;
            
            my $socket = IO::Socket::SSL->new
            (
                PeerAddr => 'market.android.com',
                PeerPort => 443,
                PeerProto => 'tcp', 
                TimeOut => 5
            );
            
            if($socket)
            {
                my $req =
                "GET /details?id=apps_topselling_paid&cat=$cat_name&start=".($i * 25)."&num=25 HTTP/1.0\r\n".
                "Host: market.android.com\r\n".
                "User-Agent: Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.9.2)\r\n".
                "Connection: Close\r\n\r\n";
                
                print $socket $req;
                
                my $data;
                while(read $socket, my $buffer, 128)
                {
                    $data .= $buffer;
                }
                
                close $socket;
                
                if($data =~ /200 OK/)
                {
                    lock $w_lock;
                    open F, '>>', $res_file or warn $!;
                    flock F, LOCK_EX;
                    
                    print F (join $/, $data =~ /data-docid="(.+?)"/g), $/;
                    
                    flock F, LOCK_UN;
                    close F;
                }
            }
        }
    }
}
