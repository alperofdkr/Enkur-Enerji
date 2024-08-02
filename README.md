# Enkur-Enerji
const moment = require('moment-timezone');
const enums = require('../../../libs/enums');
const { Inverter, InverterLog } = require('../../../models');
const ReportService = require('../../../services/ReportService');
const AlarmService = require('../../../services/AlarmService');


const cron_function = async (startTime, endTime) => {
    try {
        // Tüm inverterları al
        const inverters = await Inverter.find({ type: enums.inverter_types.ACTIVE }).lean();

        for (const inverter of inverters) {
            await checkInverterConnection(inverter, startTime, endTime);
        }
    } catch (error) {
        console.error("Error in cron_function:", error);
    }
};
const checkInverterConnection = async (inverter, startTime, endTime) => {
    try {
        // Belirtilen zaman aralığında inverter loglarını al
        const inverterLogs = await InverterLog.find({
            inverter_id: inverter._id,
            read_time_unix: {
                $gte: startTime.toDate(),
                $lte: endTime.toDate()
            }
        }).lean();

        // Eğer hiç log yoksa inverter bağlantısız olabilir
        if (inverterLogs.length === 0) {
            await handleDisconnectedInverter(inverter, startTime, endTime);
        } else {
            // Logları kontrol ederek bağlantı durumunu analiz et
            let disconnected = false;
            for (const log of inverterLogs) {
                if (log.status === enums.inverter_status.DISCONNECTED) {
                    disconnected = true;
                    break;
                }
            }

            // Bağlantısız durumu raporla
            if (disconnected) {
                await handleDisconnectedInverter(inverter, startTime, endTime);
            }
        }
    } catch (error) {
        console.error("Error in checkInverterConnection:", error);
    }
};
const handleDisconnectedInverter = async (inverter, startTime, endTime) => {
    try {
        const readTime = endTime.toDate();
        // Alarm oluştur
        await AlarmService.createInverterDisconnectedAlarm(inverter, readTime);

        // Rapor oluştur
        await ReportService.saveInverterDisconnectedReport(inverter, readTime);
    } catch (error) {
        console.error("Error in handleDisconnectedInverter:", error);
    }
};
module.exports = {
    cron_function,
    checkInverterConnection,
    handleDisconnectedInverter
};

