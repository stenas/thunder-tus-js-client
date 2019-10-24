let CryptoJS = require("crypto-js");
import axios from './axios-base' //axios lib
import Store from '../store' // Vue Store
import userSettings from './platform' //platform JS
import imageUtils from './imagesUtils'
import helpers from './helpers'

class FileChunk{
  static slice(file, blob, offset, chunkSize){
    let reader = new FileReader();
    return new Promise((resolve, reject) => {
      reader.onerror = () => {
        reader.abort();
        reject(new DOMException("Problem parsing input file."));
      };
      reader.onload = () => {
        resolve(reader.result);
      };
      reader.readAsArrayBuffer(blob.call(file, offset,chunkSize))
    });
  }

}
export default class UploadTusMedia{
  renameFile(filename){
   let pattern=/\.[0-9a-z]+$/i;
   let uuid = this.createUUID();
   let extension = filename.match(pattern);
   return uuid+extension;
  }
  createUUID(){
    var dt = new Date().getTime();
    var uuid = 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
        var r = (dt + Math.random()*16)%16 | 0;
        dt = Math.floor(dt/16);
        return (c=='x' ? r :(r&0x3|0x8)).toString(16);
    });
    return uuid;
  }
  async uploadFiles(files){
    if(files.length){
      let media = [];
      for(let file of files){

        let blobSlice = Blob.prototype.slice;
        let chunkSize = 10485760;
        let fileSlices = [];
        let fileSize = file.size;
        let fileRename = this.renameFile(file.name);
        let offset = 0;

        //get full content of file return promise
        let fileFullChecksum = await FileChunk.slice(file,blobSlice,offset,file.size)
        .then((response) => {
          let arrayBuffer = CryptoJS.lib.WordArray.create(response);
          return CryptoJS.MD5(arrayBuffer).toString(CryptoJS.enc.Base64);//CryptoJS.enc.Base64
        })
        .catch((error) => {
          return error;
        })

        let idx = 1;
        let retry = 1;
        while(offset < fileSize){
          //slice part of file
          let fileSlice = await FileChunk.slice(file,blobSlice,offset,(offset + chunkSize))
          .then((response) => {
            let arrayBuffer = CryptoJS.lib.WordArray.create(response);
            let fileSliceChecksum =  CryptoJS.MD5(arrayBuffer).toString(CryptoJS.enc.Base64);//CryptoJS.enc.Base64
            return {buffer:response,checksum:fileSliceChecksum}
          })
          .catch((error) => {
            return error;
          })
          // send slice file to server
          let newOffset = await this.send({
            content: fileSlice.buffer,
            sliceChecksum: fileSlice.checksum,
            fileSize:fileSize,
            fileFullChecksum: fileFullChecksum,
            offset: offset,
            filename: fileRename
          })

          // 3 tentivas de envio para o mesmo offset
          if(retry >= 3 && newOffset == offset){
            return{
              message:'File upload error, upload canceled try later',
              statusCode:460
            };
          }
          else{
            retry = retry + 1;
          }

          //update offset
          offset = idx * newOffset
          if(offset === fileSize){
            //DO SOMENTHING OR RETURN SOMENTHING
            // Example create the thumbnails files and return the file or fileSrc
            let fileWithSrc = null;
            if(helpers.checkTypeMedia(file.type) == 'image'){
              fileWithSrc = await imageUtils.generateThumbnail(file)
              .then((response) => {
                return response;
              })
              .catch((error) => {
                return error;
              })
              media.push({filename:fileRename, file:fileWithSrc});
            }
            else{
              media.push({filename:fileRename, file:file});
            }
          }
        }
      }
      return media;
    }
  }
  //upload file to server
  send(params){
    return new Promise((resolve, reject) => {
      axios.patch('tus/'+ params.filename,
          params.content,
          {headers:{
            'Accept':'application/json',
            // Aplication headers
            "Authorization"        : "`Bearer ACESSO_TOKEN`",
            "Yieap-User-Agent"     :  "USER_AGENT_CLIENT",
            // Standard TUS headers
            "Content-Type"         : "application/offset+octet-stream",
            "Tus-Resumable"        : "1.0.0",
            "Upload-Offset"        : params.offset,
            "Upload-Checksum"      : 'md5 '+ params.sliceChecksum,
            // ThunderTUS headers
            "CrossCheck"           : "true",
            "Express"              : "true",
            "Upload-Length"        : params.fileSize,
            "Upload-CrossChecksum" : 'md5 '+ params.fileFullChecksum
            },
          }
        )
        .then((response) => {
          //204 => upload da parte okay
          //460 => Checksum Mismatch - 3 tentivas cancelar o envio
          //410 => checksum final do ficheiro inteiro nao bate certo - cancelar o envio
          //400 => md5 server not suport - cancelar envio
          //404 => ficheiro not exist on server - cancelar envio
          console.log(response.status )
          if(response.status === 204 ){
            resolve(response.headers['upload-offset'])
          }
          else{
            reject({
              message:response.data,
              statusCode:response.status
            })
          }
        })
        .catch((error) => {
          if(error.response.status === 460){
            resolve(error.response.headers['upload-offset'])
          }
          else{
            reject({
              message:error.response.data,
              statusCode:error.response.status
            })
          }
        });
      });
    }
}
