# intern
--> Setting up Symfony & MySQL
-->Implementing CSV Upload & Storage
-->Implementing Email Notification (Asynchronous)
-->Implementing Database Backup & Restore
-->Implementing Twitter OAuth Authentication
-->Testing with Postman & Submission Guide

1.Install Symfony
  composer create-project symfony/skeleton symfony-api
cd symfony-api
composer require symfony/web-server-bundle symfony/orm-pack symfony/mailer symfony/messenger symfony/security-bundle

2.DATABASE_URL="mysql://root:password@127.0.0.1:3306/symfony_db"
  To Run:
     php bin/console doctrine:database:create
2.1 Create User Entity
  namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity(repositoryClass="App\Repository\UserRepository")
 */
class User
{
    /** @ORM\Id @ORM\GeneratedValue @ORM\Column(type="integer") */
    private $id;

    /** @ORM\Column(type="string", length=100) */
    private $name;

    /** @ORM\Column(type="string", unique=true) */
    private $email;

    /** @ORM\Column(type="string", unique=true) */
    private $username;

    /** @ORM\Column(type="string") */
    private $address;

    /** @ORM\Column(type="string", columnDefinition="ENUM('USER', 'ADMIN')") */
    private $role;

    // Getters and setters here...
}
To RUn:
   php bin/console make:migration
php bin/console doctrine:migrations:migrate

Step 3: Implement CSV Upload API:
   namespace App\Controller;

use App\Entity\User;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Routing\Annotation\Route;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;

/**
 * @Route("/api", name="api_")
 */
class UserController extends AbstractController
{
    /**
     * @Route("/upload", name="upload", methods={"POST"})
     */
    public function uploadCSV(Request $request, EntityManagerInterface $em)
    {
        $file = $request->files->get('file');

        if (!$file) {
            return new JsonResponse(['error' => 'No file uploaded'], 400);
        }

        $handle = fopen($file->getPathname(), 'r');
        while (($data = fgetcsv($handle, 1000, ",")) !== FALSE) {
            $user = new User();
            $user->setName($data[0]);
            $user->setEmail($data[1]);
            $user->setUsername($data[2]);
            $user->setAddress($data[3]);
            $user->setRole($data[4]);

            $em->persist($user);
        }
        fclose($handle);
        $em->flush();

        return new JsonResponse(['message' => 'Users uploaded successfully']);
    }
}
Step 4: Implement Email Notification
  composer require symfony/mailer
MAILER_DSN=smtp://username:password@smtp.example.com:587
use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Mime\Email;

// Inside uploadCSV() function, after inserting users:
$email = (new Email())
    ->from('admin@example.com')
    ->to($data[1])
    ->subject('Account Created')
    ->text("Hello $data[0], your account has been created.");

$mailer->send($email);

Step 5: Implement Database Backup & Restore
   /**
 * @Route("/backup", name="backup", methods={"GET"})
 */
public function backupDatabase()
{
    $backupFile = 'backup.sql';
    exec("mysqldump -u root -p password symfony_db > $backupFile");

    return new JsonResponse(['message' => 'Backup completed', 'file' => $backupFile]);
}

5.2 Restore API

/**
 * @Route("/restore", name="restore", methods={"POST"})
 */
public function restoreDatabase(Request $request)
{
    $file = $request->files->get('file');

    if (!$file) {
        return new JsonResponse(['error' => 'No backup file provided'], 400);
    }

    exec("mysql -u root -p password symfony_db < " . $file->getPathname());

    return new JsonResponse(['message' => 'Database restored successfully']);
}

Step 6: Implement Twitter OAuth
  composer require knplabs/knp-oauth2-client-bundle league/oauth1-client
6.1 Install OAuth Bundle
  composer require knplabs/knp-oauth2-client-bundle league/oauth1-client
6.2 Configure OAuth
  knpu_oauth2_client:
    clients:
        twitter:
            type: twitter
            client_id: '%env(OAUTH_TWITTER_KEY)%'
            client_secret: '%env(OAUTH_TWITTER_SECRET)%'
            redirect_route: auth_twitter_callback
6.3 Create Twitter Auth Controlle
    namespace App\Controller;

use KnpU\OAuth2ClientBundle\Client\ClientRegistry;
use Symfony\Component\Routing\Annotation\Route;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;

class AuthController extends AbstractController
{
    /**
     * @Route("/auth/twitter", name="auth_twitter")
     */
    public function connect(ClientRegistry $clientRegistry)
    {
        return $clientRegistry->getClient('twitter')->redirect();
    }

    /**
     * @Route("/auth/twitter/callback", name="auth_twitter_callback")
     */
    public function callback(ClientRegistry $clientRegistry, Request $request)
    {
        $client = $clientRegistry->getClient('twitter');
        $user = $client->fetchUser();
        
        return $this->json([
            'username' => $user->getNickname(),
            'name' => $user->getName(),
            'email' => $user->getEmail(),
        ]);
    }
}

