# Inversion-articulatoire_2

Ce projet contiendra le code impl�ment� de ac2art en utilisant PyTorch. Il y a une partie lecture/traitement des donn�es,  et une autre apprentissage. On r�cup�re les donn�es de 4 corpus dans des formats variables, on les lit, les traite, et les homog�n�ise pour qu'elles aient le m�me format. 

Il sera divis� en deux parties :

### Pr� traitement

EFfectue le pr� taitement n�c�ssaire aux bases de donn�es dont on dispose, c'est � dire :  MOCHA-TIMIT, MNGU0, USC-TIMIT, et HASKINS. Cette partie pr� traite �galement les donn�es de zerospeech test.
Il contient un script traitement par corpus.

Le dernier corpus (zerospeech) est diff�rent des deux premiers, car il ne contient que des fichiers wav (pas de .lab, ni de .ema).

les articulateurs utiles sont dans les deux dimensions : tongue tip, tongue dorsum, tongue blade, upper lip, lower lip , lower incisor.
Les �tapes du traitement ne sont pas effectu�es toujours suivant la m�me structure, car cel� d�pend du format des donn�es en entr�es. Les actions effectu�es sont les suivantes : 

* Uniquement EMA : r�cup�rer les trajectoires EMA, conserver uniquement les articulateurs utiles (voir plus haut), les mettre dans le format d'un numpy array de taile (Kx12) ou (Kx14) si on dispose des trajectoires du velum. Interpoler les donn�es manquantes avec spline cubique. Lisser les trajectoires en leur appliquant un filtre passe bas � 30Hz. Calculer ce qu'on appelle les 4 "vocal tract" et r�arranger les donn�es pour les mettre dans le m�me format (K*18), avec des 0 pour les trajectoires non disponibles. Les 18 variables en questions sont dans l'ordre suivant ['tt_x', 'tt_y', 'td_x', 'td_y', 'tb_x', 'tb_y', 'li_x', 'li_y', 'ul_x', 'ul_y', 'll_x', 'll_y','la','pro','ttcl','tbcl','v_x','v_y'].



* Uniquement WAV :  r�cup�rer le waveform en utilisant librosa. Calcule des 13 plus grands MFCC, et des Delta et DeltaDelta pour chaque coefficient (ce qui donne 13*3 = 39 coefficients) . Prise en compte du contexte en ajoutant 5 trames MFCC avant et apr�s la trame courante (ce qui donne 39*11 = 429 coefficients). 

* Enl�ve les silences � partir des fichiers d'annotation (si disponible). On rogne les fichiers EMA , et les frames MFCC (en consid�rant que la k-�me frame mfcc correspond � la p�riode [k*sr_wav/0.01, (k+1)*sr_wav/0.01] en secondes o� sr_wav est le sampling rate du wav, et 0.01 car les frames mfcc sont d�cal�es de 10ms.

* Pour les donn�es de USC TIMIT nous n'avons pas une phrase par fichier, donc on utilise les fichiers d'annotations pour d�couper les fichiers wav et ema entre chaque phrase.

* Normalisation : on normalise les coefficients mfcc de fa�on classique en soustrayant la moyenne et en divisant par l'�cart type (moyenne et �cart type cacul� sur l'ensemble du corpus). Pour les donn�es EMA au lieu de soustraire la moyenne sur l'ensemble du corpus on soustrait la moyenne sur une fen�tre glissante, et on divise par l'�cart type de l'articulateur sur l'ensemble du corpus. On enl�ve la moyenne sur une fen�tre glissante car on s'est rendu compte que la bobine bouge ou est replac�e au cours de l'enregistrement.


Les �tapes de traitement sont les suivantes et dans le m�me ordre :

Pour chaque locuteur :
	Pour chaque �chantillon (= 1 phrase):
*  R�cup�ration de la donn�e EMA. 
*  On ne conserve que les donn�es articulatoires qui nous int�ressent
*  Interpolation pour remplacer les "NaN"
*  R�cup�ration des donn�es acoustiques .WAV
*  Extraction des 13 plus grand MFCC.
*  Lecture du fichier d'annotation qui donn�e le d�but et la fin de la phrase (hors silence)
*  On enl�ve les trames MFCC et donn�es EMA correspondant au silence
*  On sous �chantillonne les donn�es articulatoires pour avoir autant de donn�es qu'il y a de trames MFCC  
*  Calcul les coefficients dynamiques \Delta et \Delta\Delta associ� � chacun des MFCC (2 coefficients en plus par MFCC)
*  On ajoute les 5 trames MFCC pr�c�dents et suivant chaque trame.

Le pr�traitement zerospeech est plus simple : 
il n'y a que des fichiers wav � charger, calculer les MFCC, les delta et deltadelta, ajouter frames contexte, puis normaliser.

Le script sauvegarde en local dans "inversion_articulatoire_2\Donnees_pretraitees\donnees_challenge_2017\1s" un fichier numpy mfcc par phrase.

FINALEMENT NON , MAIS PEUT ETRE FAIT SIMPLEMENT EN DECOMMENTANT LA LIGNE SPLIT CORPUS : Pour le corpus MNGU0, lorsque la phrase a plus de 500 frames mfcc on divise par deux nos phrases (afin d'avoir des phrases d'� peu pr�s la m�me longueur que celles du corpus mocha), et on r�it�re.

Les pr�requis de ces scripts de pr�traitement est d'avoir bien les donn�es relatives aux corpus dans \inversion_articulatoire_2\Donnees_brutes
Les fichiers doivent �tre dans les bons dossiers (leur donner les bons noms), ce qui revient mettre les dezipper les dossier t�l�charg�s.

Pour mngu0 : ema dans "ema", phonelabels dans "phone_labels", wav dans "wav"

Pour mocha : les donn�es concernant le locuteur X (fsew0 , msak0,...) dans le dossier X. 
Pour une utt�rance (ie une phrase) on a des fichier : .EMA, .EPG,.LAB, et on ne se sert pas du fichier .EPG ni .LAR

Pour zerospeech :  dans "inversion_articulatoire_2\Donnees_brutes\ZS2017" pour le moment uniquement les fichiers d'une secondes
ils sont dans le dossier "1s" et on a un ensemble de fichiers .wav

Pour usc timit : les donn�es dezipp�es ie un dossier par speaker et pour chaque speaker un dossier "trans", un "mat", et un "wav"

Pour Haskins : les donn�es d�zipp�es ie un dossier par sepaker et pour chaque speaker un dossier "data".

Les donn�es usctimit et haskins sont fournies en format matlab, et peuvent se lire avec le module sio et scipy. 

Le traitement des donn�es est diff�rent, principalement parce que le format est diff�rent que pour mocha et MNGU0. Ici nous avons un fichier .mat et un ficher .wav pour 18 secondes de parole. Le locuteur r�cite les phrases une � une, mais les enregistrements sont rogn�s toutes les 18 secondes. Nous avons autour de 3 phrases par fichiers. Comme d�j� pr�cis� pr�c�demment, les silences ne nous int�ressent pas, alors il faut red�couper ces signaux pour ne conserver que les parties correspondant � de la parole. Il y a alors deux possibilit�s : utiliser les fichiers de transcription (.trans) fournis avec le corpus, ou bien utiliser des techniques d'analyse du speech pour d�tecter les silences. Nous avons trouv� plus simple d'utiliser les fichiers de transcription. Les trajectoires articulatoires nous sont donn�es pour les 6 articulateurs classiques ( tongue tip, tongue dorsum, tongue body, lower lip, upper lip, lower incisor ) en 3 dimensions.

Nous avons cr�er une classe pour l'objet "corpus" qui contient les param�tres utiles, et �galement des fonctions communes au traitement des 4 corpus. Ces fonctions communes sont : add_vocal_tract, smooth_data, synchro_ema_mfcc, et calculate_norm_values.

Le script de traitement parcourt 2 fois l'ensemble des enregistrements. La premi�re fois les traitements sont dans l'ordre : lecture ema, lecture wav, wav to mfcc, add_vocal_tract, smooth_ema, synchro_ema_mfcc.
Ensuite les norm values sont calcul�es et stock�es dans les attributs de l'instance corpus cr�e.
Lors du deuxi�me parcourt de l'ensemble des enregistrements, il normalise les donn�es EMA et MFCC, puis refiltre les don�es EMA. 
(Puis les phrases trop longues sont coup�es en 2.)
Enfin les phrases sont d�coup�es en 3 parties train, valid et test). Par ex dans le fichier text  "Donnees_pretraitees/fileset/speaker_train" se trouve la liste des noms des fichiers qui seront dans le training set pour ce speaker.

A l'issu du traitement les sauvegardes locales sont : 
- EMA : les donn�es EMA non normalis�es et non liss�es avec les vocal tract, et les trajectoires mauvaises mises � 0.
- MFCC : les donn�es MFCC normalis�es.
- EMA_final : les donn�es avec les vocal tract liss�es et normalis�es. Ce sont celles si que nous allons utiliser pour l'apprentissage.
- norm_values : pour chaque speaker std_ema_&speaker, moving_average_&speaker, std_mfcc_&speaker, mean_mfcc_&speaker.




Une fois les donn�es homog�n�iis�es la fonction get_fileset_name est appel�e, et g�n�re 3 listes de noms de fichiers  : train,valid,test. Ce script contient une fonction "get_fileset_names(speaker)" qui choisi al�atoirement 70% des phrases du speaker pour en faire le train set, 10% pour le validation set, et 20% le test set.
Le script sauvegarde trois fichier train_speaker.txt , test_speaker.txt, valid_speaker.txt avec la liste des noms de fichiers correspondant. 



### Apprentissage

Les mod�les cr��s se trouvent ici.

Pour le moment le mod�le g�n�ral bilstm se trouve dans "class_network.py".
La fonction d'entrainement est dans "Entrainement.py", l'utilisateur peut choisir : 
 - sur quel(s) speaker(s) il entra�ne le  mod�le (le validation set sera celui correspondant au(x) speaker(s) sur lesquels on apprend)
 - sur quel(s) speaker(s) il teste le mod�le.
- le nombre d'�pochs
- la fr�quence � laquelle on �value le score sur le validation set
- la patience (ie combien d'�pochs d'overfitting cons�cutives avant d'arr�ter l'apprentissage)
- le learning rate
- si on veut que les donn�es (d'apprentissage et de test) soient filtr�es
- si on veut que le mod�le lisse les donn�es avec une couche de convolution aux poids ajust�s.
- si on veut sauvegarder des  graphes de trajectoires originales et pr�dites par le mod�le sur le test set.


Dans notre code nous avons choisi qu'une epoch correspond � un nombre d'it�rations de telle sorte que toutes les phrases aient �t� prises en compte ( 1 epoch = (n_phrase/batch_size) it�ration).
A chaque it�ration on selectionne al�atoirement &batchsize phrases. Si il y a plusieurs speakers sur lesquels apprendre, alors la probabilit� de tirer une phrase d'un speaker est proportionnelle au nombre de 
phrases prononc�es par ce speaker. Le script qui retourne les phrases s�lectionn�es se trouve dans Apprentissage\utils.py, et s'appelle load_filenames(train_on,batch_size,part) o� part est train valid ou test.
Dans ce m�me script utils.py la fonction "load_data(filenames, filtered=False)" retourne (x,y) deux listes contenant les mfcc/ema correspondant � chacun des filenames.

Dans ce script est cr�e la classe "my_bilstm" qui est cod�e en pytorch.
Elle contient deux couches denses � 300 neurones, puis deux couche Bi lstm � 300 neurones dans chaque direction, ensuite il y a une couche convolutionnelle optionnelle qui agit comme un filtre passe bas � fr�quence 30Hz si les donn�es
�taient �chantillonn�es � une fr�quence pr�cise (pr�cis�e par l'utilisateur ou par d�faut 200Hz). Ceci peut poser probl�me quand nos donn�es d'apprentissages ne sont pas toutes �chantillon�es � la m�me fr�quence.
Les poids de la convolution sont pour le moment fix�s de la mani�re suivante : on veut que la sortie soit temporelle convolu�e avec une s�quence dont la TF est une porte entre 0 et f_cutoff. Il s'agirait d'un sinus cardinal � support infini. 
Comme notre entr�e est � support fini on rogne le support du sinus cardinal en le multipliant en temporel avec une fen�tre de Hanning.
Pour le moment les poids de la convolutions sont fix�s [� suivre : faire varier fc, ou m�me un fc par articulateur].

Le script Entrainement_3 s'ajuste aux articulations "correctes". Chaque batch ne contient que des phrases homog�nes en terme d'articulateurs. Nous avons regroup� les speakers en cat�gorie, de telle sorte qu'au sein d'une cat�gorie tous les speakers aient les m�mes articulateurs corrects. 



 