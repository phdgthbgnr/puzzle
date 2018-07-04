# puzzle
## Jeu de puzzle + worker

### logique principale

````js
(function(){
            
                var _i = 0, // a remettre à zero pour chaque rechragement des js/css
                _m, _isMobile = false, _safariMobile = false,
                _curpiece, // id piece selectionnée
                _curNodepiece, // node duplique
                _isTouch,
                _droppable = null,
                _dropTouchcoord = {}, 
                _curtargetTouch = null, // id derniere piece survolee
                _initialXY = {x:0,y:0}, // coordonnes initiales de la piece selectionnee
                _curPuzzle = 1,  // level puzzle
                _maxPuzzle = 5, // nb de puzzle à compléter
                _repPiece = '_img/'+_curPuzzle+'/', // rep des pieces de puzzle à charger
                _winH = 0, // hauteur window en mode landscape 
                _offsetDragX  = 0, // desktop : position x de la souris dans l'element à draguer
                _offsetDragY  = 0, // idem y
                _started = false,
                _timer, // interval Timer
                _curTime = 0, // nb secondes (pour un puzzle)
                _totTime = 0, // nb secondes duree jeux (affichage finale)
                _stringTime = '00:00', // time elapsed for display in popup
                _scoref = 12, // nb pieces à déposer
                _score = 0, // nb pieces déposées
                _dropOK = false, // controler les drop echoués
                _ieVers = 0,
                _worker,
                _nodePuzz, // node list des cibles

                deferjs1 = [
                '_css/dbspuzzle.css',
                '_js/manageEvents14.js'
                ],
                
                progressLoadImg = function(cur, tot){
                    var nodelevel =  _m.$dc('level');
                    if(_m.hasAclass('levels','invisible')) _m.removeAclass('levels','invisible'); 
                    var bounds = _m.$dc('levels').getBoundingClientRect();
                    var wdth = (bounds.width/tot)*cur;
                    nodelevel.style.width = wdth+"px";
                    if(Math.floor(parseInt(nodelevel.style.width)) == Math.floor(bounds.width)) {
                        _m.addAclass('levels','invisible');
                    } 
                }

                deferImages = function(callback, dt, rep, progress){
                    var totImgDefer = 0;
                    var imgDefer = document.getElementsByTagName('img');
                    var nbDefer = [];
                    // dt == data-srcslide
                    for (var i=0; i<imgDefer.length; i++) {
                        // if (m.hasAclass(imgDefer[i],'nomobile') && ismobile == 1 && ipad == 0) continue;
                        if(imgDefer[i].getAttribute(dt)){
                            nbDefer.push(imgDefer[i]);
                        }
                    }
                    for (var i = 0; i<nbDefer.length; i++) {
                        try{
                            nbDefer[i].setAttribute('src', rep+nbDefer[i].getAttribute(dt));
                        }catch(err){
                            continue;
                        }

                        if (nbDefer[i].addEventListener != undefined){
                            nbDefer[i].onload = function(e){
                                if(progress) progress(totImgDefer,nbDefer.length-1);
                                if(totImgDefer >= nbDefer.length-1) {
                                    callback();
                                }
                                totImgDefer++;
                            };
                        }else if (nbDefer[i].readyState){ // IE8
                            nbDefer[i].onreadystatechange = function(){
                                if(nbDefer[i].readyState == 'loaded' || nbDefer[i].readyState == 'complete') {
                                    if(progress) progress(totImgDefer,nbDefer.length-1);
                                    if(totImgDefer >= nbDefer.length-1) {
                                        callback();
                                    }
                                    totImgDefer++;
                                }
                            };
                        }
                    
                    };
                },

                testScore = function(){
                    _score++;
                    if(_score == _scoref){
                        _curPuzzle++;
                        _m.addAclass('grille','invisible');
                        if(_curPuzzle <= _maxPuzzle){
                            // next level
                            _m.$dc('scoretime').innerHTML = _stringTime;
                            reSize(); // bug safri ipad
                            _m.removeAclass('nextlevel','nodisplay');
                            setTimeout(function(){
                                _m.addAclass('popup1','anime');
                            },100);
                            stopTimer();
                            _score=0;
                        }else{
                            // end game
                            _score = 0;
                            _curPuzzle = 1,  // level puzzle
                            _repPiece = '_img/' + _curPuzzle + '/',
                            reSize(); // bug safri ipad
                            _m.removeAclass('endgame','nodisplay');
                            setTimeout(function(){
                                _m.addAclass('popup2','anime');
                            },100);
                            var nodelevels = document.querySelectorAll('.levelb');
                            nodelevels.forEach(function(el,i,arr){
                                _m.removeAclass(el,'played');
                            });
                            _m.$dc('scoretimef').innerHTML = timeInMnSec(_totTime);
                            _totTime = 0;
                            stopTimer();
                        }
                    }

                },

                dragstartPiece = function(e,c,t){
                    return false;
                },

                onMouseDown = function(e,c,t) {
                    _curNodepiece = e.currentTarget;
                    _curpiece = e.currentTarget.id;
                    _initialXY.x = e.currentTarget.style.left;
                    _initialXY.y = e.currentTarget.style.top;
                    var img = e.currentTarget.querySelector('img');
                    var bounds = img.getBoundingClientRect();
                    if(e.targetTouches) {
                        _offsetDragX = e.touches[0].clientX-bounds.left;
                        _offsetDragY = e.touches[0].clientY-bounds.top;
                    }else{
                        var x = e.x === undefined ? e.clientX : e.x;
                        var y = e.y === undefined ? e.clientY : e.y;
                        _offsetDragX = x-bounds.left;
                        _offsetDragY = y-bounds.top;

                    }

                    // reajustement de z-index
                    var oldzindex = _curNodepiece.style.zIndex;
                    var nodespiece = document.querySelectorAll('.piece');
                    nodespiece.forEach(function(el,i,arr){
                        if(parseInt(el.style.zIndex) >= parseInt(oldzindex)) {
                            el.style.zIndex = el.style.zIndex-1;
                        }
                    });
                    _m.addAclass(_curNodepiece, 'dropshadow');

                    _curNodepiece.style.zIndex = nodespiece.length;
                    
                    _curNodepiece.style.width = bounds.width+"px";
                    _curNodepiece.style.height = bounds.height+"px";
                    
                    if(e.changedTouches || _isTouch) {
                        _m.listenerAdd(_curpiece,'touchmove', onMouseMove, true);
                    }else{
                        moveAt(_curNodepiece, e.pageX, e.pageY);
                        document.addEventListener('mousemove', onMouseMove);
                    }
                },

                moveAt = function(piece, pageX, pageY) {
                    piece.style.left = pageX - _offsetDragX + 'px';
                    piece.style.top = pageY - _offsetDragY + 'px';
                },
                
                onMouseMove = function(e,c,t) {
                    // moveAt(_curNodepiece, e.pageX, e.pageY);
                    var x,y, xc, yc;
                    if(e.targetTouches) {
                        x = e.targetTouches[0].pageX;
                        y = e.targetTouches[0].pageY;
                        xc = e.touches[0].clientX;
                        yc = e.touches[0].clientY;
                    }else{
                        x = e.pageX;
                        y = e.pageY;
                        xc = e.x === undefined ? e.clientX : e.x;
                        yc = e.y === undefined ? e.clientY : e.y;
                    }

                    _curNodepiece.style.left = x - _offsetDragX + 'px';
                    _curNodepiece.style.top  = y - _offsetDragY + 'px';

                    // safari ne peut pas passer getBoundingClientRect() en param
                    var bounds = {x:0, y:0, w:0, h:0};
                    // bounds.x = _curNodepiece.getBoundingClientRect();
                    bounds.x = _curNodepiece.getBoundingClientRect().left;
                    bounds.y = _curNodepiece.getBoundingClientRect().top;
                    bounds.w = _curNodepiece.getBoundingClientRect().width;
                    bounds.h = _curNodepiece.getBoundingClientRect().height;

                    _worker.postMessage(['testover', bounds]);
                    
                    _curNodepiece.style.visibility = 'hidden';
                    var elemBelow = document.elementFromPoint(xc, yc);
                    _curNodepiece.style.visibility = 'visible';
                    // _curNodepiece.hidden = false;
                    
                    if (!elemBelow) return;
                    
                    var droppableBelow = elemBelow.closest('.droppable');
                    if (_droppable != droppableBelow) {
                        if (_droppable) { // null when we were not over a droppable before this event
                            leaveDroppable(_droppable);
                        }
                        _droppable = droppableBelow;
                        if (_droppable) { // null if we're not coming over a droppable now
                        // (maybe just left the droppable)
                            enterDroppable(_droppable);
                        }
                    }
                },

                enterDroppable = function(elem) {
                    if(_curtargetTouch == null || _curtargetTouch == elem) _m.addAclass(elem,'over');
                    // console.log('enterDroppable');
                },
                
                leaveDroppable = function(elem) {
                    _m.removeAclass(elem,'over');
                },

                onMouseUp = function(e,c,t) {
                    // console.log('onMouseUp');
                    var n1 = e.currentTarget.id.substr(5);

                    var n2 = _curtargetTouch !== null && _curtargetTouch !== false ? _curtargetTouch.substr(5) : _curtargetTouch;
                    
                    if(n1==n2){
                        //e.currentTarget.setAttribute('src',_curpImage);
                        _m.removeAclass(_curtargetTouch,'over');
                        // _m.addAclass(_m.$dc(_curtargetTouch).children[0],'invisible');
                        _m.addAclass(_curtargetTouch,'invisible');
                        _m.addAclass(_curpiece,'nodisplay');
                        // maj object position des pieces
                        _dropTouchcoord[_curtargetTouch].reveal = true;
                        // maj worker
                        _worker.postMessage(['init',_dropTouchcoord]);
                        testScore();
                    }else{
                        _nodePuzz.forEach(function(el,i,arr){
                            _m.removeAclass(el,'over');
                        });
                        // up sur le puzzle à la mauvaise place
                        // placement à la position initiale
                        if(_curtargetTouch !== null || _curtargetTouch === false){
                            _curNodepiece.style.left = _initialXY.x;
                            _curNodepiece.style.top  = _initialXY.y;
                        }

                    }
                    if(e.targetTouches){
                        // console.log(e.currentTarget);
                        _m.listenerRemove(_curpiece, 'touchmove', true);
                    }else{
                        document.removeEventListener('mousemove', onMouseMove);
                    }
                    
                    _m.removeAclass(e.currentTarget,'dropshadow');

                    return false;
                },



                /*
                TOUCH
                */

                startTouch = function(e,c,t){
                    _curpiece = e.currentTarget.id;
                    _curNodepiece = e.currentTarget;
                    // _m.$dc(_curpiece).setAttribute('style','z-index:10000');
                    _m.listenerAdd(_curpiece,'touchmove', moveTouch, true);
                    // var event = new TouchEvent('touchstart');
                    // console.log(_m.$dc('zonedrop01').dispatchEvent(event));
                    return false;
                },

                endTouch = function(e,c,t){
                    var n1 = _curpiece.substr(5);
                    // var id = e.currentTarget.getAttribute('id');
                    // console.log(id);
                    var n2 = _curtargetTouch.substr(5);
                    if(n1==n2){
                        //e.currentTarget.setAttribute('src',_curpImage);
                        _m.removeAclass(_curtargetTouch,'over');
                        _m.addAclass(_m.$dc(_curtargetTouch).children[0],'nodisplay');
                        _m.addAclass(_curpiece,'nodisplay');
                    }
                    _m.listenerRemove(_curpiece, 'touchmove', true);
                    return false;
                },

                moveTouch = function(e,c,t){
                    
                },

                is_touch_device = function() {
                    return (('ontouchstart' in window)
                    || (navigator.MaxTouchPoints > 0)
                    || (navigator.msMaxTouchPoints > 0));
                },

                whenAllPiecesLoaded = function(){
                    var nodesdropp = document.querySelectorAll('.imgdroppable');
                    // mise en tableau des dimensions et coordonnées des cible de puzzle pour test hit
                    nodesdropp.forEach(function(el,n,l){
                        // console.log(el.parentNode.id);
                        var Bounds = el.getBoundingClientRect();
                        // console.log(Bounds);
                        _dropTouchcoord[el.parentNode.id] = {'x':Bounds.left, 'y':Bounds.top, 'w':Bounds.width, 'h':Bounds.height, 'reveal': false};
                        // console.log(w);
                    });
                    console.log(_dropTouchcoord)
                    // passe l'objet au worker
                    _worker.postMessage(['init',_dropTouchcoord]);

                    var bounds1 = _m.$dc('blocpieces').getBoundingClientRect();
                    var bounds2 = _m.$dc('gui').getBoundingClientRect();
                    var nodespieces = document.querySelectorAll('.piece');
                    nodespieces.forEach(function(el,i,arr){
                        var id = el.getAttribute('id');
                        var ii = i < 9 ? '0'+(i+1): i+1;
                        var zdrop = 'zdrop'+ii;
                        // mise à la taille des pieces de puzzle (== taille des cibles)
                        _m.$dc(id).children[0].style.width  =  _dropTouchcoord[zdrop]['w']+"px";
                        _m.$dc(id).children[0].style.height =  _dropTouchcoord[zdrop]['h']+"px";

                        // position aléatoire des pieces
                        var x1 = bounds1.left
                        var x2 = bounds1.right - (_dropTouchcoord[zdrop]['w']+10);
                        var y1 = bounds2.height;
                        var y2 = bounds1.height - (_dropTouchcoord[zdrop]['h']+10);
                        var x = Math.floor(Math.random()*(x2-x1+1)+x1); 
                        var y = Math.floor(Math.random()*(y2-y1+1)+y1);  
                        _m.$dc(id).style.left = x+"px";
                        _m.$dc(id).style.top  = y+"px";
                    });
                    // affiche niveau
                    _m.addAclass('level'+_curPuzzle,'played');
                    return false;
                },

                clicPlay = function(e,c,t){
                    if(_started) return false;
                    _started = true;
                    _m.removeAclass('play','anime');
                    var st1;
                    deferImages(function(){
                        st1 = setTimeout(function(){
                            _m.removeAclass('contener','nodisplay');

                            if(!_isTouch){
                                // _m.listenClass('piece', 'mousedown', downPiece, false);
                                _m.listenClass('piece', 'dragstart', dragstartPiece, false); // false
                                _m.listenClass('piece', 'mousedown', onMouseDown, true);
                                _m.listenClass('piece', 'mouseup', onMouseUp, true);
                            }else{
                                _m.listenClass('piece', 'touchstart', onMouseDown, true);
                                _m.listenClass('piece', 'touchend', onMouseUp, true);
                                // _m.listenClass('droppable', 'mouseenter', overMouse, true);
                                // _m.listenerAdd('zonedrop01', 'touchstart', overMouse, false);
                            }
                            
                            // defer image bg puzzle
                            deferImages(function(){
                                _m.addAclass('intro','nodisplay');
                               
                                // bug IE11 -----------------
                                if(_ieVers < 15) {
                                    var wp = _m.$dc('placement').getBoundingClientRect().width;
                                    _m.$dc('board').style.width = wp+"px";
                                }
                                // ---------------------------
                                reSize();
                                whenAllPiecesLoaded();
                                startTimer();
                                clearTimeout(st1);
                                st1 = null;
                            },'data-bg','');
                            
                        },200)
                    }, 'data-src', _repPiece, progressLoadImg);
                    return false;
                },

                nextLevel = function(e,c,t){
                    _repPiece = '_img/'+_curPuzzle+'/';
                    _m.removeAclass('grille','invisible');
                    // _m.addAclass('puzzle','nodisplay');
                    _m.$dc('bgpuzzle').setAttribute('data-bg','_img/bg0'+_curPuzzle+'.jpg');
                    
                    var nodesDrop = document.querySelectorAll('.droppable');
                    nodesDrop.forEach(function(el,i,arr){
                        _m.removeAclass(el,'invisible');
                    });
                    // bug IE11 -----------------
                    if(_ieVers < 15) {
                        var wp = _m.$dc('placement').style.width;
                        _m.$dc('puzzle').style.width = wp;
                    }
                    // ---------------------------
                    deferImages(function(){
                        console.log('images deferred 2');
                        var nodesPiece = document.querySelectorAll('.piece');
                        nodesPiece.forEach(function(el,i,arr){
                            _m.removeAclass(el,'nodisplay');
                        });
                        // _m.removeAclass('puzzle','nodisplay');
                        whenAllPiecesLoaded();
                        deferImages(function(){
                            if(_m.hasAclass('popup1','anime')) _m.removeAclass('popup1','anime');
                            setTimeout(function(){
                                if(!_m.hasAclass('nextlevel','nodisplay')) _m.addAclass('nextlevel','nodisplay');
                            },250);
                            if(!_m.hasAclass('endgame','nodisplay')) _m.addAclass('endgame','nodisplay');
                            
                            startTimer();
                        },'data-bg','');
                    },'data-src', _repPiece, progressLoadImg);
                    return false;
                },

                startTimer = function(){
                    _curTime = 0;
                    _timer = setInterval(function(){
                        _curTime++;
                        _totTime++;
                        _m.$dc('curtime').innerHTML = timeInMnSec(_curTime); 
                    },1000);
                },

                stopTimer = function(){
                    clearInterval(_timer);
                    _m.$dc('curtime').innerHTML = '00:00';
                },

                timeInMnSec = function(t){
                    var sec = t/60 < 1 ? t : t%60;
                    var mn = t > 60 ? Math.floor(t/60) : '0';
                    sec = sec < 10 ? '0'+sec : sec;
                    mn = mn < 10 ? '0'+mn : mn;
                    _stringTime = mn +':' + sec;
                    return _stringTime;
                },

                initAfter = function(){
                    _m = manageEvents;
                    _m.removeAclass('levels','nodisplay');
                    _m.addAclass('play','anime');
                    _m.removeAclass('portrait','nodisplay');
                    deferImages(function(){
                        _m.removeAclass('intro','nodisplay');
                        _m.listenerAdd('play', 'click', clicPlay, true);
                        _m.listenerAdd('continue','click', nextLevel, true);
                        _m.listenerAdd('replay','click', nextLevel, true);
                        var t = setTimeout(function(){
                            // obligation d'un délai : les transitions et les display:none -> block
                            _m.addAclass('logo','appear');
                        },100);
                    },'data-src1','', progressLoadImg);
                    
                    //worker
                    _nodePuzz = document.querySelectorAll('.droppable');
                    if(window.Worker){
                        _worker = new Worker('_js/worker.js');
                        _worker.onmessage = function(e){
                            _nodePuzz.forEach(function(el,i,arr){
                                _m.removeAclass(el,'over');
                            });
                            _curtargetTouch = e.data;
                            // console.log('_curtargetTouch from worker', _curtargetTouch);
                            if(_curtargetTouch)_m.addAclass(e.data,'over');  
                        }
                    };
                    
                    _m.listenerAdd('fullscrn','click', function(e,c,t){
                        var docfscr = window.document;
                        var docElfscr = docfscr.documentElement;
                        
                        requestFullScreen = docElfscr.requestFullscreen || docElfscr.mozRequestFullScreen || docElfscr.webkitRequestFullScreen || docElfscr.msRequestFullscreen;
                        cancelFullScreen = docfscr.exitFullscreen || docfscr.mozCancelFullScreen || docfscr.webkitExitFullscreen || docfscr.msExitFullscreen;

                        if (requestFullScreen !== undefined){
                            requestFullScreen.call(docElfscr);
                            _m.addAclass('fullscrn','nodisplay');
                        }

                    },true);

                    if(!_safariMobile && _isMobile) _m.removeAclass('fullscrn','nodisplay');
                },

                reSize = function(){
                    var bounds = _m.$dc('board').getBoundingClientRect();
                    _m.$dc('nextlevel').style.left = bounds.width+"px";
                    _m.$dc('endgame').style.left = bounds.width+"px";
                },

                downloadJSAtOnload = function(arr, callback) {
                    // charge JS et CSS
                    var t = arr.length-1;
                    if (arr[_i].match('^(.*\.js)')){
                    var element = document.createElement('script');
                        element.setAttribute('type','text/javascript');
                        element.setAttribute('src',arr[_i]);
                        if (element.addEventListener != undefined){
                        element.addEventListener('load',function(e){
                            _i++;
                            if(_i <= t) downloadJSAtOnload(arr, callback);
                            if(_i > t) {
                                callback();
                            }
                        });
                        }else if (element.readyState){ // IE8
                            element.onreadystatechange = function(){
                                if(element.readyState == 'loaded' || element.readyState == 'complete') {
                                    _i++;
                                if(_i <= t) downloadJSAtOnload(arr, callback);
                                    if(_i > t) {
                                        callback();
                                    }
                                }
                            }
                        }
                        if(_i <= t) document.body.appendChild(element);
                    };
                    // chargement CSS
                    if (arr[_i].match('^(.*\.css)$')){
                        // loadStylesheet(deferjs[i]);
                        if (document.createStyleSheet){
                            document.createStyleSheet(arr[_i]);
                        }else {
                            var stylesheet = document.createElement('link');
                            stylesheet.href = arr[_i];
                            stylesheet.rel = 'stylesheet';
                            stylesheet.type = 'text/css';
                            document.getElementsByTagName('head')[0].appendChild(stylesheet);
                        }
                        _i++;
                        if(_i <= t) downloadJSAtOnload(arr, callback);
                        //if(i > t) window.myConsole('CSS chargées');
                    }
                },

                resizeWind = function(e){
                    //sortie mode plein écran (android)
                    var bound = _m.$dc('placement').getBoundingClientRect();
                    if(bound.width > 0) {
                        _m.$dc('board').style.width = bound.width+"px";
                        _m.$dc('puzzle').style.width = bound.width+"px";
                        setTimeout(function(){
                            whenAllPiecesLoaded();
                        },0);
                    }
                },

                detectSorientation = function(e){
                    // alert('detectSorientation');
                },

                //iphone / ipad
                detectWorientation = function(e){
                    if(_safariMobile){
                        var hgth = 0;
                        var wdth = 0;
                        var wwdth = window.outerWidth > 0 ? window.outerWidth : window.innerWidth;
                        var hhdth = window.outerHeight > 0 ? window.outerHeight : window.innerHeight;
                        if(screen && document.documentElement){
                            hgth = document.documentElement.clientWidth == wwdth ? (screen.height > screen.width?screen.width:screen.height): window.innerHeight;
                            hgth = window.outerHeight > 0? hgth : window.innerHeight;
                            wdth = document.documentElement.clientHeight == hhdth ? (screen.width > screen.height?screen.height:screen.width): window.innerWidth;
                            wdth = window.outerWidth > 0? wdth : window.innerWidth;
                            
                        }else{
                            hgth = window.innerHeight;
                            wdth = window.innerWidth;
                        }
                        
                        // var wp = _m.$dc('placement').getBoundingClientRect().width;
                        _m.$dc('intro').setAttribute('style','height:'+ hgth +'px; width:'+ wdth +'px');
                        _m.$dc('contener').setAttribute('style','height:'+ hgth +'px; width:'+ wdth +'px');
                        _m.$dc('blocpieces').setAttribute('style','height:'+ hgth +'px');
                        _m.$dc('nextlevel').setAttribute('style','height:'+ hgth +'px');// left:'+wp+'px');
                        _m.$dc('endgame').setAttribute('style','height:'+ hgth +'px');
                        
                        hgth+=100;
                        _m.$dc('mybody').setAttribute('style','height:'+ hgth +'px; width:'+ wdth +'px');
                        // _m.$dc('menu').setAttribute('style','padding-top:10px');
                        
                        // suppression de tous les touch inutile
                        // afin d'éviter scroll ecran en mode "fullscreen"
                        _m.listenerAdd(document,'touchmove',function(e,c,t){
                             return false;
                        },false);
                        _m.listenerAdd('gui','touchmove',function(e,c,t){
                            return false;
                        },true);
                        _m.listenerAdd('intro','touchmove',function(e,c,t){
                             return false;
                        },true);
                        _m.listenerAdd('blocpieces','touchmove',function(e,c,t){
                             return false;
                        },true);
                        _m.listenClass('droppable','touchmove',function(e,c,t){
                             return false;
                        },true);
                        _m.listenerAdd('nextlevel','touchmove',function(e,c,t){
                             return false;
                        },true);
                        _m.listenerAdd('endgame','touchmove',function(e,c,t){
                             return false;
                        },true);
                            
                    }
                    
                    window.scrollTo(0, 10);
                },

                DomLoaded = function(){
                    // if (ie9) deferjs1.push('_js/Points.js');
                    if(navigator.userAgent.match('/Edge\/18|Edge\/17|Edge\/16|Edge\/15')) _ieVers = 15;
                    if(navigator.userAgent.match('/trident\/7|rv:11')) _ieVers = 11;
                    if(ie10 || navigator.userAgent.match('/trident\/6|MSIE 10/')) _ieVers = 10;
                    if(ie9) _ieVers = 9;
                    if(ie8) _ieVers = 8;
                    if(_ieVers < 15 && _ieVers > 0) deferjs1.push('_js/foreach.polyfill.js', '_js/foreachnodelist.polyfill.js', '_js/closest.polyfill.js');
                    // if(_ieVers == 15) deferjs1.push('_js/setdragimagepolyfill.js');
                    _isTouch = is_touch_device();
                    _winH = window.innerHeight < window.innerWidth ? window.innerHeight : window.innerWidth;
                    downloadJSAtOnload(deferjs1, initAfter);

                    if (window.addEventListener){
                        window.addEventListener('resize', resizeWind); // passage on / off fullscreen
                    }else if (window.attachEvent){ // IE8
                        window.attachEvent('onresize', resizeWind);
                    }else{
                        window.onreisze = resizeWind();
                    };

                };

                if (window.addEventListener){
                    window.addEventListener('DOMContentLoaded', function(){DomLoaded()});
                    // window.addEventListener('resize', resizeWind); // passage on / off fullscreen
                }else if (window.attachEvent){ // IE8
                    window.attachEvent('onload', function() { DomLoaded(); });
                    // window.attachEvent('onresize', resizeWind);
                }else{
                    window.onload = DomLoaded();
                    // window.onreisze = resizeWind();
                };

                if(navigator.userAgent.match(/Tablet|iPad|Mobile|Windows Phone|Lumia|Android|webOS|iPhone|iPod|Blackberry|PlayBook|BB10|Opera Mini|\bCrMo\/|Opera Mobi/i)) _isMobile = true;
                if(navigator.userAgent.match(/Safari/i) && !navigator.userAgent.match(/Chrome/i)) _safariMobile = true;

                // si window.orientation existe on l'utilise de preference
                if (window.addEventListener && (screen.orientation !== undefined && screen.orientation !== null)){
                //if (screen.orientation) window.addEventListener('orientationchange', detectSorientation, false);
                window.addEventListener('orientationchange', detectSorientation, false);
                }else if (window.orientation !== undefined && window.orientation !== null){
                window.addEventListener('orientationchange', detectWorientation, false);
                //}else if (window.orientation !== undefined && window.orientation !== null) {
                //window.onorientationchange = detectWorientation;
                }else if (window.attachEvent && window.addEventListener === undefined) {// IE8
                if (screen.orientation) window.attachEvent('onorientationchange', detectSorientation);
                if (window.orientation && (screen.orientation !== undefined || screen.orientation !== null)) window.attachEvent('onorientationchange', detectWorientation);
                };

            })();
````

### Worker (gestion des overlap)

````js
this._wdropTouchcoord = {};
onmessage = function(e) {
    switch(e.data[0]){
        case 'init':
        this._wdropTouchcoord = e.data[1];
        // console.log('ok ', _wdropTouchcoord);
        break;
        case 'testover':
        // console.log('testover', e.data[1].x, e.data[1].y, e.data[1].w, e.data[1].h);
        var m = e.data[1],
        // offset = 10;
        ret = null,
        allowDrop = false, // test si tous les emplacements sont remplis (noAllowDrop = true)
        d1x = m.x,
        d1y = m.y,
        d1xMax = m.x+m.w,
        d1yMax = m.y+m.h,
        xyoverlap = 0,
        overlap = 0,
        inSurface = 0,
        testFull = 0;
        
        for(var i in this._wdropTouchcoord){
            // allowDrop = false;
            k = this._wdropTouchcoord[i];
            var d2x = k.x, 
            d2y = k.y, 
            d2xMax = k.x+k.w,
            d2yMax = k.y+k.h,
            // calcul overlap
            x_overlap = Math.max(0, Math.min(d1xMax,d2xMax) - Math.max(d1x,d2x))
            y_overlap = Math.max(0, Math.min(d1yMax,d2yMax) - Math.max(d1y,d2y));

            overlap = x_overlap*y_overlap;

            if(overlap > 0) inSurface = 1;
           
            if(overlap > 0 && overlap > xyoverlap && k.reveal == false){
                xyoverlap = overlap;
                ret = i;
            }
        }
        // pas dans la surface on renvoie : null
        if(inSurface == 0) ret = null; // inutile : pour debug
        // dans la surface mais emplacement occupé : false
        if((inSurface == 1) && (ret === null)) ret = false;

        postMessage(ret)
        break;
    }
    // postMessage(workerResult);
  }

  ````
