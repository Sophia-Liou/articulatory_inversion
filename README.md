# Inversion-articulatoire_2

Ce projet contiendra le code impl�ment� de ac2art en utilisant PyTorch.

Il sera divis� en deux parties :

### Pr� traitement

EFfectue le pr� taitement n�c�ssaire aux bases de donn�es dont on dispose, c'est � dire :  MOCHA-TIMIT, MNGU0, USC-TIMIT, et HASKINS. Cette partie pr� traite �galement les donn�es de zerospeech test.
Il contient un script traitement par corpus.

Le dernier corpus (zerospeech) est diff�rent des deux premiers, car il ne contient que des fichiers wav (pas de .lab, ni de .ema).

les articulateurs utils sont dans les deux dimensions : tongue tip, tongue dorsum, tongue blade, upper lip, lower lip , lower incisor.
Les �tapes du traitement ne sont pas effectu�es toujours suivant la m�me structure, car cel� d�pend du format des donn�es en entr�es. Les actions effectu�es sont les suivantes : 

* Uniquement EMA : r�cup�rer les trajectoires EMA, conserver uniquement les articulateurs utiles (voir plus haut), les mettre dans le format d'un numpy array de taile (Kx12) ou (Kx14) si on dispose des trajectoires du velum. Interpoler les donn�es manquantes avec spline cubique. Lisser les trajectoires en leur appliquant un filtre passe bas � 30Hz.

* Uniquement WAV :  r�cup�rer le waveform en utilisant librosa. Calcule des 13 plus grands MFCC, et des Delta et DeltaDelta pour chaque coefficient (ce qui donne 13*3 = 39 coefficients) . Prise en compte du contexte en ajoutant 5 trames MFCC avant et apr�s la trame courante (ce qui donne 39*11 = 429 coefficients). 

* Pour les donn�es de USC TIMIT nous n'avons pas une phrase par fichier, donc on utilise les fichiers d'annotations pour d�couper les fichiers wav et ema entre chaque phrase.

* Normalisation : on normalise les coefficients mfcc de fa�on classique en soustrayant la moyenne et en divisant par l'�cart type (moyenne et �cart type cacul� sur l'ensemble du corpus). Pour les donn�es EMA au lieu de soustraire la moyenne sur l'ensemble du corpus on soustrait la moyenne sur une fen�tre glissante, et on divise par l'�cart type de l'articulateur MAXIMUM parmis tous les articulateurs. Les raisons de cette normalisation sont expliqu�es plus bas.




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


Une fois qu'on a charg� en m�moire toutes les phrases :
*  Calcul de la moyenne et �cart type par articulation/coefficient acoustique (EMA ou Delta Feature),

	Pour chaque �chantillon (=1 phrase):
*  Normalisation par rapport � l'ensemble du corpus pour le m�me speaker ==> enregistrement d'un fichier .npy dans "Donnees_pretraitees/speaker/ema(oumfcc)/nomfichier.npy"
*  flitrage passe bas avec sinc+fen�tre de hanning (� 25Hz pour MNGU0 et 30Hz pour mocha) des donn�es ema ==> enregistrement d'un fichier .npy dans "Donnees_pretraitees/speaker/ema_filtered/nomfichier.npy"


Le script sauvegarde en local dans inversion_articulatoire_2\Donnees_pretraitees\Donnees_breakfast\MNGU0 deux fichiers numpy par phrase.
Un fichier pour les mfcc (dans \mfcc) et un fichier ema (dans \ema)

Le pr�traitement zerospeech est plus simple : 
il n'y a que des fichiers wav � charger, calculer les MFCC, les delta et deltadelta, ajouter frames contexte, puis normaliser.

Le script sauvegarde en local dans "inversion_articulatoire_2\Donnees_pretraitees\donnees_challenge_2017\1s" un fichier numpy mfcc par phrase.

Pour le corpus MNGU0, lorsque la phrase a plus de 500 frames mfcc on divise par deux nos phrases (afin d'avoir des phrases d'� peu pr�s la m�me longueur que celles du corpus mocha), et on r�it�re.

Les pr�requis de ces scripts de pr�traitement est d'avoir bien les donn�es relatives aux corpus dans \inversion_articulatoire_2\Donnees_brutes\Donnees_breakfast
Les fichiers doivent �tre dans les bons dossiers, ce qui revient mettre les dezipper les dossier t�l�charg�s.
Pour mngu0 : ema dans "ema_basic_data", phonelabels dans "phone_labels", wav dans "wav_16KHz"
pour mocha : les donn�es concernant le locuteur X (fsew0 ou msak0) dans le dossier X. 
Pour une utt�rance (ie une phrase) on a des fichier : .EMA, .EPG,.LAB, .LAR, et on ne se sert pas du fichier .EPG.
Pour zerospeech :  dans "inversion_articulatoire_2\Donnees_brutes\donnees_challenge_2017" pour le moment uniquement les fichiers d'une secondes
ils sont dans le dossier "1s" et on a un ensemble de fichiers .wav

Apr�s avoir fait trourner "traitement_mocha","traitement_mngu0", et "traitement_zerospeech", nous avons pour chaque phrase :
 1 fichier .NPY pour les mfcc, et 1 fichier .NPY pour les donn�es EMA (sauf pour ZS2017)


###  Create filesets

Ce script contient une fonction "get_fileset_names(speaker)" qui choisi al�atoirement 70% des phrases du speaker pour en faire le train set, 10% pour le validation set, et 20% le test set.
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



 